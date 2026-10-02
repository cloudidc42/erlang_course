# Part 63: Search Engine Implementation

> **"Search is the discovery layer of every great product"**  
> Search คือ discovery layer ของทุก product ที่ยิ่งใหญ่

---

## สารบัญ

1. [Inverted Index](#1-inverted-index)
2. [Tokenizer and Analyzer](#2-tokenizer-and-analyzer)
3. [TF-IDF Ranking](#3-tf-idf-ranking)
4. [Full-Text Search Engine](#4-full-text-search-engine)
5. [Autocomplete](#5-autocomplete)
6. [Faceted Search](#6-faceted-search)
7. [แบบฝึกหัด](#7-แบบฝึกหัด)

---

## 1. Inverted Index

```erlang
%% inverted_index.erl — inverted index using ETS
-module(inverted_index).
-export([new/0, add_document/3, remove_document/2, search/2, search_phrase/2]).

-define(INDEX_TABLE, search_index).
-define(DOCS_TABLE,  search_docs).

new() ->
    ets:new(?INDEX_TABLE, [named_table, set, public,
                           {read_concurrency, true}]),
    ets:new(?DOCS_TABLE, [named_table, set, public,
                          {read_concurrency, true}]),
    ok.

%% Add document: index all terms
add_document(DocId, Title, Body) ->
    %% Store original document
    ets:insert(?DOCS_TABLE, {DocId, #{title => Title, body => Body}}),
    %% Tokenize and index
    Terms = tokenize(<<Title/binary, " ", Body/binary>>),
    TermFreqs = term_frequencies(Terms),
    TotalTerms = length(Terms),
    maps:foreach(fun(Term, Freq) ->
        %% posting: {doc_id, term_freq, doc_length, position_list}
        Positions = positions_of(Term, Terms),
        update_posting(Term, DocId, Freq, TotalTerms, Positions)
    end, TermFreqs).

update_posting(Term, DocId, Freq, DocLen, Positions) ->
    Posting = #{doc_id => DocId, freq => Freq,
                doc_len => DocLen, positions => Positions},
    case ets:lookup(?INDEX_TABLE, Term) of
        [{Term, Postings}] ->
            ets:insert(?INDEX_TABLE, {Term, [Posting | Postings]});
        [] ->
            ets:insert(?INDEX_TABLE, {Term, [Posting]})
    end.

remove_document(DocId, _Body) ->
    ets:delete(?DOCS_TABLE, DocId),
    %% Remove from all postings
    ets:foldl(fun({Term, Postings}, _) ->
        NewPostings = [P || P <- Postings, maps:get(doc_id, P) =/= DocId],
        case NewPostings of
            [] -> ets:delete(?INDEX_TABLE, Term);
            _  -> ets:insert(?INDEX_TABLE, {Term, NewPostings})
        end
    end, ok, ?INDEX_TABLE).

%% Boolean AND search: documents containing all terms
search(Query, MaxResults) ->
    Terms = tokenize(Query),
    case Terms of
        [] -> [];
        _  ->
            PostingLists = [get_postings(T) || T <- Terms],
            DocSets = [sets:from_list([maps:get(doc_id, P) || P <- PL])
                       || PL <- PostingLists],
            CommonDocs = lists:foldl(fun sets:intersection/2,
                                     hd(DocSets), tl(DocSets)),
            %% Rank and return top N
            AllPostings = lists:flatten(PostingLists),
            Scored = score_documents(sets:to_list(CommonDocs),
                                     Terms, AllPostings),
            SortedIds = [DocId || {_, DocId} <- lists:reverse(lists:sort(Scored))],
            Results = [get_document(DocId) || DocId <- lists:sublist(SortedIds, MaxResults)],
            [R || R =/= undefined, R <- Results]
    end.

%% Phrase search: terms must appear adjacent
search_phrase(Phrase, MaxResults) ->
    Terms = tokenize(Phrase),
    case Terms of
        [] -> [];
        _  ->
            PostingLists = [get_postings(T) || T <- Terms],
            DocIds = phrase_match(PostingLists),
            [get_document(DocId) || DocId <- lists:sublist(DocIds, MaxResults)]
    end.

phrase_match([]) -> [];
phrase_match([First | Rest]) ->
    FirstDocs = [{maps:get(doc_id, P), maps:get(positions, P)} || P <- First],
    lists:filtermap(fun({DocId, Positions}) ->
        case check_phrase(DocId, Positions, Rest, 1) of
            true  -> {true, DocId};
            false -> false
        end
    end, FirstDocs).

check_phrase(_DocId, _Positions, [], _Offset) -> true;
check_phrase(DocId, Positions, [NextPostings | Rest], Offset) ->
    NextForDoc = [P || P <- NextPostings, maps:get(doc_id, P) =:= DocId],
    case NextForDoc of
        [] -> false;
        [P | _] ->
            NextPositions = maps:get(positions, P),
            ValidPositions = [Pos || Pos <- Positions,
                                     lists:member(Pos + Offset, NextPositions)],
            case ValidPositions of
                [] -> false;
                _  -> check_phrase(DocId, ValidPositions, Rest, Offset + 1)
            end
    end.

get_postings(Term) ->
    case ets:lookup(?INDEX_TABLE, Term) of
        [{Term, Postings}] -> Postings;
        [] -> []
    end.

get_document(DocId) ->
    case ets:lookup(?DOCS_TABLE, DocId) of
        [{DocId, Doc}] -> Doc#{id => DocId};
        [] -> undefined
    end.

positions_of(Term, Terms) ->
    [Pos || {Pos, T} <- lists:zip(lists:seq(0, length(Terms)-1), Terms),
            T =:= Term].
```

---

## 2. Tokenizer and Analyzer

```erlang
%% text_analyzer.erl — tokenization, stemming, stop words
-module(text_analyzer).
-export([tokenize/1, normalize/1, stem/1, is_stopword/1]).

-define(STOP_WORDS, [
    <<"a">>, <<"an">>, <<"the">>, <<"and">>, <<"or">>, <<"but">>,
    <<"in">>, <<"on">>, <<"at">>, <<"to">>, <<"for">>, <<"of">>,
    <<"with">>, <<"by">>, <<"from">>, <<"is">>, <<"are">>, <<"was">>,
    <<"were">>, <<"be">>, <<"been">>, <<"being">>, <<"have">>, <<"has">>
]).

tokenize(Text) when is_binary(Text) ->
    %% Lowercase, split on non-alphanumeric
    Lower = string:lowercase(Text),
    Words = re:split(Lower, "[^a-z0-9]+", [{return, binary}, trim]),
    %% Filter stop words and short tokens
    Filtered = [W || W <- Words,
                     byte_size(W) >= 2,
                     not is_stopword(W)],
    %% Stem each word
    [stem(W) || W <- Filtered].

normalize(Term) ->
    string:lowercase(Term).

is_stopword(Word) ->
    lists:member(Word, ?STOP_WORDS).

%% Simple Porter stemmer suffix rules (subset for demonstration)
stem(Word) when byte_size(Word) =< 4 -> Word;
stem(Word) ->
    apply_rules(Word, [
        %% Plural endings
        {<<"ies">>, <<"y">>},
        {<<"ied">>, <<"y">>},
        {<<"ness">>, <<>>},
        {<<"ment">>, <<>>},
        {<<"ing">>, <<>>},
        {<<"tion">>, <<>>},
        {<<"ed">>,  <<>>},
        {<<"er">>,  <<>>},
        {<<"ly">>,  <<>>},
        {<<"s">>,   <<>>}
    ]).

apply_rules(Word, []) -> Word;
apply_rules(Word, [{Suffix, Replacement} | Rest]) ->
    SufLen = byte_size(Suffix),
    WordLen = byte_size(Word),
    case WordLen > SufLen + 2 of  %% keep at least 3 chars
        true ->
            case binary:part(Word, WordLen - SufLen, SufLen) of
                Suffix ->
                    Stem = binary:part(Word, 0, WordLen - SufLen),
                    <<Stem/binary, Replacement/binary>>;
                _ ->
                    apply_rules(Word, Rest)
            end;
        false ->
            apply_rules(Word, Rest)
    end.

%% Term frequencies
term_frequencies(Terms) ->
    lists:foldl(fun(Term, Acc) ->
        maps:update_with(Term, fun(N) -> N+1 end, 1, Acc)
    end, #{}, Terms).

tokenize(Text) ->
    text_analyzer:tokenize(Text).
```

---

## 3. TF-IDF Ranking

```erlang
%% tfidf.erl — TF-IDF scoring
-module(tfidf).
-export([score/3, idf/2]).

%% TF-IDF = TF * IDF
%% TF = term frequency in document (normalized by doc length)
%% IDF = inverse document frequency = log(N / df(t))

score(TermFreq, DocLength, IdfScore) ->
    TF = TermFreq / DocLength,
    TF * IdfScore.

idf(NumDocs, DocFreq) when DocFreq > 0 ->
    math:log((NumDocs + 1) / DocFreq);
idf(_, 0) -> 0.

%% BM25 (better than simple TF-IDF)
bm25(TermFreq, DocLength, AvgDocLength, IdfScore) ->
    K1 = 1.5,  %% term frequency saturation
    B  = 0.75, %% length normalization
    TF_norm = TermFreq * (K1 + 1) /
              (TermFreq + K1 * (1 - B + B * DocLength / AvgDocLength)),
    IdfScore * TF_norm.

score_documents(DocIds, QueryTerms, AllPostings) ->
    NumDocs = ets:info(search_docs, size),
    [begin
         TermScores = [score_term(DocId, Term, AllPostings, NumDocs)
                       || Term <- QueryTerms],
         TotalScore = lists:sum(TermScores),
         {TotalScore, DocId}
     end || DocId <- DocIds].

score_term(DocId, Term, AllPostings, NumDocs) ->
    DocPostings = [P || P <- AllPostings,
                        maps:get(doc_id, P) =:= DocId],
    TermPostings = [P || P <- AllPostings,
                         maps:get(doc_id, P) =:= DocId],
    case [P || P <- TermPostings,
               maps:get(doc_id, P) =:= DocId] of
        [] -> 0.0;
        [P | _] ->
            Freq    = maps:get(freq, P),
            DocLen  = maps:get(doc_len, P),
            DocFreq = length([X || X <- AllPostings,
                                   maps:get(doc_id, X) =/= undefined]),
            IdfVal  = idf(NumDocs, max(DocFreq, 1)),
            AvgLen  = avg_doc_length(),
            _ = DocPostings,
            bm25(Freq, DocLen, AvgLen, IdfVal)
    end.

avg_doc_length() ->
    case ets:info(search_docs, size) of
        0 -> 100;
        N ->
            TotalLen = ets:foldl(fun({_, #{body := B}}, Acc) ->
                Acc + length(binary:split(B, <<" ">>, [global]))
            end, 0, search_docs),
            TotalLen div N
    end.
```

---

## 4. Full-Text Search Engine

```erlang
%% search_engine.erl — complete search engine GenServer
-module(search_engine).
-behaviour(gen_server).
-export([start_link/0, index/3, search/2, search_phrase/2, suggest/2,
         remove/1, stats/0]).
-export([init/1, handle_call/3, handle_cast/2]).

-record(state, {
    index   :: ets:tab(),
    docs    :: ets:tab(),
    trie    :: ets:tab()   %% for autocomplete
}).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

index(DocId, Title, Body) ->
    gen_server:cast(?MODULE, {index, DocId, Title, Body}).

search(Query, Opts) ->
    MaxResults = maps:get(limit, Opts, 10),
    gen_server:call(?MODULE, {search, Query, MaxResults}).

search_phrase(Phrase, Opts) ->
    MaxResults = maps:get(limit, Opts, 10),
    gen_server:call(?MODULE, {search_phrase, Phrase, MaxResults}).

suggest(Prefix, Limit) ->
    gen_server:call(?MODULE, {suggest, Prefix, Limit}).

remove(DocId) ->
    gen_server:cast(?MODULE, {remove, DocId}).

stats() ->
    gen_server:call(?MODULE, stats).

init([]) ->
    Index = ets:new(search_index, [set, {read_concurrency, true}]),
    Docs  = ets:new(search_docs,  [set, {read_concurrency, true}]),
    Trie  = ets:new(search_trie,  [set, {read_concurrency, true}]),
    {ok, #state{index=Index, docs=Docs, trie=Trie}}.

handle_call({search, Query, Max}, _From, State) ->
    Results = inverted_index:search(Query, Max),
    Highlighted = [highlight(R, Query) || R <- Results],
    {reply, {ok, Highlighted}, State};

handle_call({suggest, Prefix, Limit}, _From, #state{trie=Trie} = State) ->
    Suggestions = trie_prefix_search(Trie, Prefix, Limit),
    {reply, {ok, Suggestions}, State};

handle_call(stats, _From, #state{index=I, docs=D} = State) ->
    {reply, #{
        total_docs  => ets:info(D, size),
        total_terms => ets:info(I, size)
    }, State}.

handle_cast({index, DocId, Title, Body}, #state{trie=Trie} = State) ->
    inverted_index:add_document(DocId, Title, Body),
    %% Update trie for autocomplete
    Words = text_analyzer:tokenize(Title),
    [trie_insert(Trie, W) || W <- Words],
    {noreply, State};

handle_cast({remove, DocId}, State) ->
    case ets:lookup(search_docs, DocId) of
        [{DocId, #{body := Body}}] ->
            inverted_index:remove_document(DocId, Body);
        [] -> ok
    end,
    {noreply, State}.

%% Highlight matching terms in snippet
highlight(Doc, Query) ->
    Terms = text_analyzer:tokenize(Query),
    Body  = maps:get(body, Doc, <<>>),
    Snippet = extract_snippet(Body, Terms, 200),
    Doc#{snippet => Snippet, highlighted_terms => Terms}.

extract_snippet(Body, Terms, MaxLen) ->
    %% Find first occurrence of a query term
    Lower = string:lowercase(Body),
    case find_first_term(Lower, Terms) of
        not_found ->
            binary:part(Body, 0, min(MaxLen, byte_size(Body)));
        Pos ->
            Start = max(0, Pos - 50),
            Length = min(MaxLen, byte_size(Body) - Start),
            Excerpt = binary:part(Body, Start, Length),
            case Start > 0 of
                true  -> <<"...", Excerpt/binary, "...">>;
                false -> <<Excerpt/binary, "...">>
            end
    end.

find_first_term(_Body, []) -> not_found;
find_first_term(Body, [Term | Rest]) ->
    case binary:match(Body, Term) of
        {Pos, _} -> Pos;
        nomatch  -> find_first_term(Body, Rest)
    end.

trie_insert(Trie, Word) ->
    Prefixes = [binary:part(Word, 0, N) || N <- lists:seq(1, byte_size(Word))],
    [ets:insert(Trie, {P, Word}) || P <- Prefixes].

trie_prefix_search(Trie, Prefix, Limit) ->
    Matches = ets:match(Trie, {Prefix, '$1'}),
    UniqueWords = lists:usort(lists:flatten(Matches)),
    lists:sublist(UniqueWords, Limit).
```

---

## 5. Autocomplete

```erlang
%% autocomplete.erl — fast prefix completion
-module(autocomplete).
-export([new/0, insert/2, complete/3, complete_ranked/3]).

%% Trie node: #{char => child_trie, is_end => bool, freq => N}
new() ->
    #{children => #{}, is_end => false, freq => 0}.

insert(Trie, Word) ->
    insert(Trie, Word, 1).

insert(Trie, <<>>, _Freq) ->
    Trie#{is_end => true,
          freq => maps:get(freq, Trie, 0) + 1};
insert(Trie = #{children := Children}, <<C:8, Rest/binary>>, Freq) ->
    Child = maps:get(C, Children, new()),
    NewChild = insert(Child, Rest, Freq),
    Trie#{children => Children#{C => NewChild}}.

complete(Trie, Prefix, MaxResults) ->
    case navigate_to(Trie, Prefix) of
        not_found -> [];
        SubTrie   -> collect_words(SubTrie, Prefix, MaxResults, [])
    end.

navigate_to(Trie, <<>>) -> Trie;
navigate_to(#{children := Children}, <<C:8, Rest/binary>>) ->
    case maps:find(C, Children) of
        {ok, Child} -> navigate_to(Child, Rest);
        error       -> not_found
    end.

collect_words(_, _, 0, Acc) -> Acc;
collect_words(#{is_end := true, children := Children, freq := F}, Prefix, Max, Acc) ->
    Acc1 = [{F, Prefix} | Acc],
    collect_from_children(Children, Prefix, Max - 1, Acc1);
collect_words(#{children := Children}, Prefix, Max, Acc) ->
    collect_from_children(Children, Prefix, Max, Acc).

collect_from_children(Children, Prefix, Max, Acc) when Max > 0 ->
    maps:fold(fun(Char, Child, {M, A}) ->
        NewPrefix = <<Prefix/binary, Char>>,
        Words = collect_words(Child, NewPrefix, M, A),
        {M - length(Words) + length(A), Words}
    end, {Max, Acc}, Children),
    Acc;
collect_from_children(_, _, _, Acc) -> Acc.

complete_ranked(Trie, Prefix, MaxResults) ->
    Words = complete(Trie, Prefix, MaxResults * 2),
    Sorted = lists:reverse(lists:sort(Words)),
    [W || {_, W} <- lists:sublist(Sorted, MaxResults)].
```

---

## 6. Faceted Search

```erlang
%% faceted_search.erl — search with facets (categories, filters)
-module(faceted_search).
-export([search_with_facets/3, build_facets/2]).

%% Search results with aggregated facets
search_with_facets(Query, Filters, Opts) ->
    %% First do full-text search
    BaseResults = inverted_index:search(Query, 1000),
    %% Apply filters
    Filtered = apply_filters(BaseResults, Filters),
    %% Build facets from filtered results
    Facets = build_facets(Filtered, maps:get(facet_fields, Opts, [])),
    %% Paginate
    Limit  = maps:get(limit, Opts, 10),
    Offset = maps:get(offset, Opts, 0),
    Page   = lists:sublist(lists:nthtail(Offset, Filtered), Limit),
    #{
        results     => Page,
        total       => length(Filtered),
        facets      => Facets,
        query       => Query
    }.

apply_filters(Results, Filters) ->
    maps:fold(fun(Field, Value, Acc) ->
        [R || R <- Acc, matches_filter(R, Field, Value)]
    end, Results, Filters).

matches_filter(Doc, Field, Value) when is_list(Value) ->
    %% Multi-value filter: match any
    DocValue = maps:get(Field, Doc, undefined),
    lists:member(DocValue, Value);
matches_filter(Doc, Field, {range, Min, Max}) ->
    DocValue = maps:get(Field, Doc, 0),
    DocValue >= Min andalso DocValue =< Max;
matches_filter(Doc, Field, Value) ->
    maps:get(Field, Doc, undefined) =:= Value.

build_facets(Results, FacetFields) ->
    maps:from_list([
        {Field, count_facet_values(Results, Field)}
        || Field <- FacetFields
    ]).

count_facet_values(Results, Field) ->
    Counts = lists:foldl(fun(Doc, Acc) ->
        Value = maps:get(Field, Doc, <<"unknown">>),
        maps:update_with(Value, fun(N) -> N+1 end, 1, Acc)
    end, #{}, Results),
    %% Return sorted by count desc
    Sorted = lists:reverse(lists:sort(
        [{Count, Value} || {Value, Count} <- maps:to_list(Counts)])),
    [#{value => V, count => C} || {C, V} <- Sorted].
```

---

## 7. แบบฝึกหัด

1. เพิ่ม synonym expansion: ค้นหา "car" ให้ return ผลของ "automobile" ด้วย
2. Implement fuzzy search: แก้ typo ด้วย edit distance (Levenshtein)
3. เพิ่ม field boosting: title match = 3x weight vs body match
4. สร้าง search index ที่ persist ลง disk ด้วย DETS

---

## สรุป Part 63

✅ Inverted index ด้วย ETS + posting lists  
✅ Tokenizer พร้อม stop words และ stemming  
✅ TF-IDF และ BM25 scoring algorithms  
✅ Full-text search engine GenServer  
✅ Trie-based autocomplete ด้วย prefix search  
✅ Faceted search พร้อม dynamic aggregations  

---

*Part 63/100 | [← ก่อนหน้า](../part62/README.md) | [ถัดไป →](../part64/README.md)*
