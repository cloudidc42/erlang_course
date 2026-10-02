# Part 78: Social Network Backend

> **"A social network is a system for managing relationships at scale"**  
> โซเชียลเน็ตเวิร์กคือระบบจัดการความสัมพันธ์ในระดับขนาดใหญ่

---

## สารบัญ

1. [Social Graph Architecture](#1-social-graph-architecture)
2. [Follow/Friend System](#2-followfriend-system)
3. [Activity Feed (Fan-out)](#3-activity-feed-fan-out)
4. [Post and Comment System](#4-post-and-comment-system)
5. [Notification System](#5-notification-system)
6. [Trending and Discovery](#6-trending-and-discovery)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Social Graph Architecture

```
Social Network System Architecture
════════════════════════════════════════════════════════

CORE PROBLEMS:
  1. Graph storage: follow/friend relationships (billions of edges)
  2. Feed generation: "what should user X see?" (fan-out)
  3. Real-time: notifications, presence, typing indicators
  4. Discovery: who to follow, trending content

ERLANG STRENGTHS FOR SOCIAL:
  ✓ Per-user gen_server: each user is an isolated process
  ✓ pg groups: broadcast to followers efficiently
  ✓ ETS: in-memory feed cache (hot users)
  ✓ WebSocket: real-time delivery
  ✓ Lightweight processes: millions of connected users

SCALE PATTERNS:
  Hot users (>1M followers): "celebrity problem"
    → Fan-out on read (pull from user's posts)
    → Not fan-out on write (would write 1M+ feed entries)

  Normal users (<10k followers): fan-out on write
    → Push to all followers' feeds on post creation
    → Fast reads from pre-computed feed
```

---

## 2. Follow/Friend System

```erlang
%% social_graph.erl — manage follower/following relationships
-module(social_graph).
-export([follow/2, unfollow/2, get_followers/1, get_following/1,
         is_following/2, mutual_follows/2]).

follow(FollowerId, FolloweeId) ->
    case FollowerId =:= FolloweeId of
        true  -> {error, cannot_follow_self};
        false ->
            Sql = "INSERT INTO follows (follower_id, followee_id, created_at)
                   VALUES ($1, $2, NOW())
                   ON CONFLICT DO NOTHING",
            case db:execute(Sql, [FollowerId, FolloweeId]) of
                {ok, 1} ->
                    increment_counter(FolloweeId, followers_count, 1),
                    increment_counter(FollowerId,  following_count, 1),
                    event_bus:publish(social_events, #{
                        type        => user_followed,
                        follower_id => FollowerId,
                        followee_id => FolloweeId
                    }),
                    ok;
                {ok, 0} ->
                    {error, already_following};
                {error, _} = Err -> Err
            end
    end.

unfollow(FollowerId, FolloweeId) ->
    Sql = "DELETE FROM follows WHERE follower_id = $1 AND followee_id = $2",
    case db:execute(Sql, [FollowerId, FolloweeId]) of
        {ok, 1} ->
            increment_counter(FolloweeId, followers_count, -1),
            increment_counter(FollowerId,  following_count, -1),
            ok;
        {ok, 0} ->
            {error, not_following}
    end.

get_followers(UserId) ->
    get_followers(UserId, #{limit => 20, cursor => undefined}).

get_followers(UserId, Opts) ->
    Limit  = maps:get(limit, Opts, 20),
    Cursor = maps:get(cursor, Opts, undefined),
    {WhereClause, Params} = cursor_clause(Cursor, UserId),
    Sql = iolist_to_binary([
        "SELECT u.id, u.username, u.display_name, u.avatar_url, f.created_at
         FROM follows f JOIN users u ON u.id = f.follower_id
         WHERE f.followee_id = $1", WhereClause,
        " ORDER BY f.created_at DESC LIMIT $2"
    ]),
    {ok, Rows} = db:query(Sql, Params ++ [Limit]),
    format_users(Rows).

get_following(UserId) ->
    Sql = "SELECT u.id, u.username, u.display_name, u.avatar_url
           FROM follows f JOIN users u ON u.id = f.followee_id
           WHERE f.follower_id = $1 ORDER BY f.created_at DESC",
    {ok, Rows} = db:query(Sql, [UserId]),
    format_users(Rows).

is_following(FollowerId, FolloweeId) ->
    %% Cache in ETS for performance
    CacheKey = {following, FollowerId, FolloweeId},
    case ets:lookup(social_cache, CacheKey) of
        [{_, Result, Exp}] when Exp > erlang:system_time(second) ->
            Result;
        _ ->
            Sql = "SELECT 1 FROM follows WHERE follower_id=$1 AND followee_id=$2",
            Result = case db:query(Sql, [FollowerId, FolloweeId]) of
                {ok, [_]} -> true;
                {ok, []}  -> false
            end,
            ets:insert(social_cache,
                       {CacheKey, Result, erlang:system_time(second) + 60}),
            Result
    end.

mutual_follows(UserIdA, UserIdB) ->
    Sql = "SELECT 1 FROM follows
           WHERE follower_id = $1 AND followee_id = $2",
    {ok, AFollowsB} = db:query(Sql, [UserIdA, UserIdB]),
    {ok, BFollowsA} = db:query(Sql, [UserIdB, UserIdA]),
    length(AFollowsB) > 0 andalso length(BFollowsA) > 0.

cursor_clause(undefined, UserId) -> {<<>>, [UserId]};
cursor_clause(Cursor, UserId) ->
    {<<" AND f.created_at < $3">>, [UserId, Cursor]}.

increment_counter(UserId, Field, Delta) ->
    Sql = iolist_to_binary([
        "UPDATE users SET ", atom_to_binary(Field), " = ", atom_to_binary(Field),
        " + $2 WHERE id = $1"
    ]),
    db:execute(Sql, [UserId, Delta]).

format_users(Rows) ->
    [#{id => Id, username => Username, display_name => Name,
       avatar_url => Avatar} || {Id, Username, Name, Avatar} <- Rows].
```

---

## 3. Activity Feed (Fan-out)

```erlang
%% feed_service.erl — activity feed with fan-out on write
-module(feed_service).
-behaviour(gen_server).

-export([start_link/0, push_to_feeds/2, get_feed/2, mark_read/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(FEED_SIZE, 200).    % max items per user feed
-define(HOT_THRESHOLD, 50000).  % followers count for "celebrity" mode

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

%% Called when a user creates a post
push_to_feeds(AuthorId, PostId) ->
    case get_followers_count(AuthorId) of
        Count when Count > ?HOT_THRESHOLD ->
            %% Celebrity: don't fan-out, followers will pull when they open app
            logger:info("Skipping fan-out for celebrity ~p (~p followers)",
                        [AuthorId, Count]),
            ok;
        _ ->
            %% Normal user: fan-out to all followers
            gen_server:cast(?MODULE, {fanout, AuthorId, PostId})
    end.

get_feed(UserId, Opts) ->
    Limit  = maps:get(limit, Opts, 20),
    Cursor = maps:get(cursor, Opts, undefined),
    %% Check in-memory cache first
    case get_cached_feed(UserId, Limit, Cursor) of
        {ok, Items} -> {ok, Items};
        miss        -> load_feed_from_db(UserId, Limit, Cursor)
    end.

mark_read(UserId, LastSeenId) ->
    db:execute("UPDATE feeds SET seen = TRUE WHERE user_id = $1 AND post_id <= $2",
               [UserId, LastSeenId]).

init([]) -> {ok, #{}}.

handle_cast({fanout, AuthorId, PostId}, State) ->
    %% Get all follower IDs
    {ok, FollowerIds} = get_all_follower_ids(AuthorId),
    %% Batch insert into feed table
    batch_insert_feeds(FollowerIds, PostId),
    {noreply, State}.

handle_call(_Req, _From, State) -> {reply, ok, State}.

batch_insert_feeds([], _PostId) -> ok;
batch_insert_feeds(FollowerIds, PostId) ->
    %% Insert in chunks of 1000
    {Chunk, Rest} = split_chunk(FollowerIds, 1000),
    %% Use unnest for efficient bulk insert
    Sql = "INSERT INTO feeds (user_id, post_id, created_at)
           SELECT unnest($1::uuid[]), $2, NOW()
           ON CONFLICT DO NOTHING",
    db:execute(Sql, [Chunk, PostId]),
    batch_insert_feeds(Rest, PostId).

split_chunk(List, N) when length(List) =< N -> {List, []};
split_chunk(List, N) -> lists:split(N, List).

get_cached_feed(_UserId, _Limit, _Cursor) -> miss.

load_feed_from_db(UserId, Limit, undefined) ->
    Sql = "SELECT p.id, p.content, p.created_at, u.username, u.avatar_url
           FROM feeds f
           JOIN posts p ON p.id = f.post_id
           JOIN users u ON u.id = p.author_id
           WHERE f.user_id = $1
           ORDER BY f.created_at DESC LIMIT $2",
    {ok, Rows} = db:query(Sql, [UserId, Limit]),
    {ok, format_feed(Rows)};
load_feed_from_db(UserId, Limit, Cursor) ->
    Sql = "SELECT p.id, p.content, p.created_at, u.username, u.avatar_url
           FROM feeds f
           JOIN posts p ON p.id = f.post_id
           JOIN users u ON u.id = p.author_id
           WHERE f.user_id = $1 AND f.created_at < $3
           ORDER BY f.created_at DESC LIMIT $2",
    {ok, Rows} = db:query(Sql, [UserId, Limit, Cursor]),
    {ok, format_feed(Rows)}.

format_feed(Rows) ->
    [#{id => Id, content => Content, created_at => CreatedAt,
       author => #{username => Username, avatar_url => Avatar}}
     || {Id, Content, CreatedAt, Username, Avatar} <- Rows].

get_all_follower_ids(AuthorId) ->
    Sql = "SELECT follower_id FROM follows WHERE followee_id = $1",
    case db:query(Sql, [AuthorId]) of
        {ok, Rows} -> {ok, [Id || {Id} <- Rows]};
        Err        -> Err
    end.

get_followers_count(AuthorId) ->
    case db:query("SELECT followers_count FROM users WHERE id = $1", [AuthorId]) of
        {ok, [{Count}]} -> Count;
        _               -> 0
    end.
```

---

## 4. Post and Comment System

```erlang
%% posts.erl — create and manage posts with reactions
-module(posts).
-export([create/2, get/1, delete/2, react/3, unreact/3, get_reactions/1]).

create(AuthorId, Attrs) ->
    PostId = generate_id(),
    Sql = "INSERT INTO posts (id, author_id, content, media_urls, created_at)
           VALUES ($1, $2, $3, $4, NOW())",
    Media = maps:get(media_urls, Attrs, []),
    case db:execute(Sql, [PostId, AuthorId, maps:get(content, Attrs), Media]) of
        {ok, _} ->
            %% Fan out to followers
            feed_service:push_to_feeds(AuthorId, PostId),
            {ok, PostId};
        Err -> Err
    end.

get(PostId) ->
    Sql = "SELECT p.*, u.username, u.avatar_url,
           (SELECT COUNT(*) FROM likes WHERE post_id = p.id) as likes_count,
           (SELECT COUNT(*) FROM comments WHERE post_id = p.id) as comments_count
           FROM posts p JOIN users u ON u.id = p.author_id
           WHERE p.id = $1 AND p.deleted_at IS NULL",
    case db:query(Sql, [PostId]) of
        {ok, [Row]} -> {ok, format_post(Row)};
        {ok, []}    -> {error, not_found}
    end.

delete(PostId, UserId) ->
    %% Soft delete
    Sql = "UPDATE posts SET deleted_at = NOW()
           WHERE id = $1 AND author_id = $2",
    case db:execute(Sql, [PostId, UserId]) of
        {ok, 1} -> ok;
        {ok, 0} -> {error, not_found_or_not_author}
    end.

react(PostId, UserId, ReactionType) ->
    Valid = [like, love, laugh, angry, sad, wow],
    case lists:member(ReactionType, Valid) of
        false -> {error, invalid_reaction};
        true  ->
            Sql = "INSERT INTO reactions (post_id, user_id, type, created_at)
                   VALUES ($1, $2, $3, NOW())
                   ON CONFLICT (post_id, user_id)
                   DO UPDATE SET type = $3",
            db:execute(Sql, [PostId, UserId, atom_to_binary(ReactionType)])
    end.

unreact(PostId, UserId, _ReactionType) ->
    Sql = "DELETE FROM reactions WHERE post_id = $1 AND user_id = $2",
    db:execute(Sql, [PostId, UserId]).

get_reactions(PostId) ->
    Sql = "SELECT type, COUNT(*) FROM reactions
           WHERE post_id = $1 GROUP BY type",
    {ok, Rows} = db:query(Sql, [PostId]),
    maps:from_list([{binary_to_atom(Type), Count} || {Type, Count} <- Rows]).

format_post({Id, AuthorId, Content, Media, CreatedAt, _, Username, Avatar,
             LikesCount, CommentsCount}) ->
    #{id => Id, author_id => AuthorId, content => Content,
      media_urls => Media, created_at => CreatedAt,
      author => #{username => Username, avatar_url => Avatar},
      likes_count => LikesCount, comments_count => CommentsCount}.

generate_id() -> base64:encode(crypto:strong_rand_bytes(12)).
```

---

## 5. Notification System

```erlang
%% notifications.erl — user notification delivery
-module(notifications).
-behaviour(gen_server).

-export([start_link/0, create/3, get_unread/1, mark_read/2]).
-export([init/1, handle_call/3, handle_cast/2]).

-define(UNREAD_CACHE_TTL, 30).  % seconds

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

create(UserId, Type, Data) ->
    gen_server:cast(?MODULE, {create, UserId, Type, Data}).

get_unread(UserId) ->
    gen_server:call(?MODULE, {get_unread, UserId}).

mark_read(UserId, NotificationIds) ->
    gen_server:call(?MODULE, {mark_read, UserId, NotificationIds}).

init([]) ->
    %% Subscribe to social events to create notifications
    lists:foreach(fun(Topic) ->
        event_bus:subscribe(Topic, self())
    end, [social_events, post_events, comment_events]),
    {ok, #{}}.

handle_cast({create, UserId, Type, Data}, State) ->
    Sql = "INSERT INTO notifications (user_id, type, data, created_at)
           VALUES ($1, $2, $3::jsonb, NOW())",
    db:execute(Sql, [UserId, atom_to_binary(Type), json:encode(Data)]),
    %% Invalidate unread cache
    ets:delete(notification_cache, {unread, UserId}),
    %% Push to connected WebSocket if online
    case ws_presence:get_connection(UserId) of
        {ok, WsPid} ->
            WsPid ! {push_notification, #{type => Type, data => Data}};
        _ -> ok
    end,
    {noreply, State}.

handle_call({get_unread, UserId}, _From, State) ->
    Now = erlang:system_time(second),
    Result = case ets:lookup(notification_cache, {unread, UserId}) of
        [{_, Notifs, Exp}] when Exp > Now -> Notifs;
        _ ->
            Sql = "SELECT id, type, data, created_at FROM notifications
                   WHERE user_id = $1 AND read_at IS NULL
                   ORDER BY created_at DESC LIMIT 50",
            {ok, Rows} = db:query(Sql, [UserId]),
            Notifs = format_notifications(Rows),
            ets:insert(notification_cache,
                       {{unread, UserId}, Notifs, Now + ?UNREAD_CACHE_TTL}),
            Notifs
    end,
    {reply, {ok, Result}, State};

handle_call({mark_read, UserId, Ids}, _From, State) ->
    Sql = "UPDATE notifications SET read_at = NOW()
           WHERE user_id = $1 AND id = ANY($2::int[])",
    db:execute(Sql, [UserId, Ids]),
    ets:delete(notification_cache, {unread, UserId}),
    {reply, ok, State}.

handle_info({event, social_events, #{type := user_followed,
             follower_id := FollowerId, followee_id := FolloweeId}}, State) ->
    create(FolloweeId, new_follower, #{follower_id => FollowerId}),
    {noreply, State};

handle_info({event, post_events, #{type := comment_added,
             post_author_id := AuthorId, commenter_id := CommenterId,
             post_id := PostId}}, State)
  when AuthorId =/= CommenterId ->
    create(AuthorId, comment_on_post,
           #{commenter_id => CommenterId, post_id => PostId}),
    {noreply, State};

handle_info(_, State) -> {noreply, State}.

format_notifications(Rows) ->
    [#{id => Id, type => binary_to_atom(Type),
       data => json:decode(Data), created_at => CreatedAt}
     || {Id, Type, Data, CreatedAt} <- Rows].
```

---

## 6. Trending and Discovery

```erlang
%% trending.erl — compute trending topics and users
-module(trending).
-behaviour(gen_server).

-export([start_link/0, get_trending_posts/1, get_trending_hashtags/0,
         get_suggested_users/2]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

-define(TRENDING_WINDOW, 3600).   % 1 hour window
-define(REFRESH_INTERVAL, 60000). % refresh every minute

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

get_trending_posts(Limit) ->
    gen_server:call(?MODULE, {trending_posts, Limit}).

get_trending_hashtags() ->
    gen_server:call(?MODULE, trending_hashtags).

get_suggested_users(UserId, Limit) ->
    gen_server:call(?MODULE, {suggested_users, UserId, Limit}).

init([]) ->
    self() ! refresh,
    {ok, #{trending_posts => [], trending_hashtags => []}}.

handle_info(refresh, _State) ->
    NewState = #{
        trending_posts    => compute_trending_posts(),
        trending_hashtags => compute_trending_hashtags()
    },
    erlang:send_after(?REFRESH_INTERVAL, self(), refresh),
    {noreply, NewState}.

handle_call({trending_posts, Limit}, _From, State) ->
    Posts = lists:sublist(maps:get(trending_posts, State), Limit),
    {reply, {ok, Posts}, State};

handle_call(trending_hashtags, _From, State) ->
    {reply, {ok, maps:get(trending_hashtags, State)}, State};

handle_call({suggested_users, UserId, Limit}, _From, State) ->
    %% "People you may know": friends-of-friends, mutual interests
    Sql = "
        SELECT u.id, u.username, u.display_name, u.avatar_url,
               COUNT(DISTINCT m.id) as mutual_follows
        FROM users u
        JOIN follows f2 ON f2.followee_id = u.id
        JOIN follows f3 ON f3.followee_id = f2.follower_id
                        AND f3.follower_id = $1
        LEFT JOIN follows m ON m.follower_id = $1 AND m.followee_id = u.id
        WHERE u.id != $1
          AND NOT EXISTS (
            SELECT 1 FROM follows WHERE follower_id = $1 AND followee_id = u.id
          )
        GROUP BY u.id, u.username, u.display_name, u.avatar_url
        ORDER BY mutual_follows DESC
        LIMIT $2
    ",
    {ok, Rows} = db:query(Sql, [UserId, Limit]),
    Users = [#{id => Id, username => Username, display_name => Name,
               avatar_url => Avatar, mutual_follows => Mutual}
             || {Id, Username, Name, Avatar, Mutual} <- Rows],
    {reply, {ok, Users}, State}.

handle_cast(_Msg, State) -> {noreply, State}.

compute_trending_posts() ->
    Sql = "
        SELECT p.id, p.content, COUNT(r.id) + COUNT(c.id) * 2 as score
        FROM posts p
        LEFT JOIN reactions r ON r.post_id = p.id
                              AND r.created_at > NOW() - INTERVAL '1 hour'
        LEFT JOIN comments  c ON c.post_id = p.id
                              AND c.created_at > NOW() - INTERVAL '1 hour'
        WHERE p.created_at > NOW() - INTERVAL '24 hours'
          AND p.deleted_at IS NULL
        GROUP BY p.id, p.content
        ORDER BY score DESC
        LIMIT 50
    ",
    {ok, Rows} = db:query(Sql, []),
    [#{id => Id, content => Content, score => Score}
     || {Id, Content, Score} <- Rows].

compute_trending_hashtags() ->
    Sql = "
        SELECT hashtag, COUNT(*) as count
        FROM post_hashtags ph
        JOIN posts p ON p.id = ph.post_id
        WHERE p.created_at > NOW() - INTERVAL '1 hour'
        GROUP BY hashtag
        ORDER BY count DESC
        LIMIT 20
    ",
    {ok, Rows} = db:query(Sql, []),
    [#{hashtag => Tag, count => Count} || {Tag, Count} <- Rows].
```

---

## 7. แบบฝึกหัด

1. Implement "lists" feature: ผู้ใช้สามารถสร้าง curated lists ของ accounts
2. เพิ่ม mentions (@username) ที่ trigger notification ไปยังผู้ถูกกล่าวถึง
3. สร้าง block/mute system ที่กรองออกจาก feed และ notifications
4. Implement "close friends" feed: เห็นเฉพาะ content จาก VIP list

---

## สรุป Part 78

✅ Social graph: follow/unfollow with counter updates, cursor pagination  
✅ Activity feed: fan-out on write for normal users, pull for celebrities  
✅ Posts and reactions: create/delete, 6 reaction types, counts  
✅ Notification system: event-driven creation, WebSocket push, cache  
✅ Trending: score-based ranking, hashtags, friends-of-friends suggestions  

---

*Part 78/100 | [← ก่อนหน้า](../part77/README.md) | [ถัดไป →](../part79/README.md)*
