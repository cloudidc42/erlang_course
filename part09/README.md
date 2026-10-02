# Part 09: I/O และ File Operations

> **"File I/O in Erlang is clean and functional — always handle errors"**  
> File I/O ใน Erlang สะอาดและ functional — จัดการ errors เสมอ

---

## สารบัญ

1. [Standard I/O](#1-standard-io)
2. [File Operations พื้นฐาน](#2-file-operations-พื้นฐาน)
3. [Reading Files](#3-reading-files)
4. [Writing Files](#4-writing-files)
5. [File Metadata](#5-file-metadata)
6. [Directory Operations](#6-directory-operations)
7. [Random Access I/O](#7-random-access-io)
8. [Streaming Large Files](#8-streaming-large-files)
9. [Binary File I/O](#9-binary-file-io)
10. [Path Operations](#10-path-operations)
11. [File Permissions](#11-file-permissions)
12. [แบบฝึกหัด](#12-แบบฝึกหัด)

---

## 1. Standard I/O

### io Module

```erlang
%% Output
1> io:format("Hello, World!~n").
Hello, World!
ok

2> io:format("Name: ~s, Age: ~p~n", ["Alice", 30]).
Name: Alice, Age: 30
ok

3> io:format(standard_error, "Error: ~p~n", [something_wrong]).
Error: something_wrong
ok

%% Input
4> io:get_line("Enter your name: ").
Enter your name: Alice
"Alice\n"

5> Name = string:trim(io:get_line("Name: "), trailing, "\n").

%% Read term
6> io:read("Enter a term: ").
Enter a term: {ok, 42}.
{ok, {ok, 42}}

%% Read characters
7> io:get_chars("Enter chars: ", 5).
Enter chars: hello
"hello"
```

### Output Formatting

```erlang
%% io:format vs io_lib:format
%% io:format    -> แสดงผลเลย
%% io_lib:format -> คืน iolist (ใช้ build string)

1> io:format("~p~n", [hello]).
hello
ok

2> Str = io_lib:format("~p", [hello]).
[[104,101,108,108,111]]  %% iolist

3> iolist_to_binary(Str).
<<"hello">>

%% Formatted output ไปยัง device ต่างๆ
io:format(standard_io, "~p~n", [term]).
io:format(standard_error, "Error: ~p~n", [error]).
io:format(File, "~p~n", [term]).  %% ไปยังไฟล์ที่เปิดแล้ว
```

---

## 2. File Operations พื้นฐาน

### เปิดและปิดไฟล์

```erlang
%% file:open/2
%% Modes: read, write, append, raw, binary, read_ahead, delayed_write
%%         {encoding, Encoding}, exclusive, sync

%% เปิดเพื่ออ่าน
{ok, File} = file:open("test.txt", [read]).
file:close(File).

%% เปิดเพื่อเขียน
{ok, File} = file:open("output.txt", [write]).
file:close(File).

%% เปิดแบบ binary
{ok, File} = file:open("data.bin", [read, binary]).

%% เปิดแบบ append
{ok, File} = file:open("log.txt", [append]).

%% Error handling
case file:open("missing.txt", [read]) of
    {ok, File} ->
        %% use file
        file:close(File);
    {error, enoent} ->
        io:format("File not found~n");
    {error, eacces} ->
        io:format("Permission denied~n");
    {error, Reason} ->
        io:format("Error: ~p~n", [Reason])
end.
```

### file:consult/1

```erlang
%% อ่านไฟล์ที่มี Erlang terms (config files)
%% File: config.erl
%% {port, 8080}.
%% {host, "localhost"}.
%% {debug, true}.

{ok, Terms} = file:consult("config.erl").
%% [{port, 8080}, {host, "localhost"}, {debug, true}]

%% parse config
parse_config(File) ->
    case file:consult(File) of
        {ok, Terms} ->
            {ok, maps:from_list(Terms)};
        {error, {Line, Mod, Error}} ->
            Msg = Mod:format_error(Error),
            {error, {syntax_error, Line, Msg}};
        {error, Reason} ->
            {error, {file_error, Reason}}
    end.
```

---

## 3. Reading Files

### อ่านทั้งไฟล์

```erlang
%% อ่าน binary ทั้งไฟล์ (เร็วที่สุด)
{ok, Binary} = file:read_file("test.txt").

%% อ่านเป็น string
{ok, Bin} = file:read_file("test.txt"),
Content = binary_to_list(Bin).

%% อ่านเป็น lines
read_lines(File) ->
    {ok, Bin} = file:read_file(File),
    binary:split(Bin, <<"\n">>, [global]).

%% อ่าน Erlang terms
{ok, Terms} = file:consult("terms.erl").
```

### อ่านทีละบรรทัด

```erlang
%% เปิดและอ่านทีละบรรทัด
process_file_lines(Path) ->
    {ok, File} = file:open(Path, [read]),
    try
        read_lines_loop(File, [])
    after
        file:close(File)
    end.

read_lines_loop(File, Acc) ->
    case io:get_line(File, "") of
        eof ->
            lists:reverse(Acc);
        {error, Reason} ->
            throw({read_error, Reason});
        Line ->
            read_lines_loop(File, [string:trim(Line)|Acc])
    end.

%% อ่านและ process ทีละบรรทัด (memory efficient)
process_each_line(Path, Fun) ->
    {ok, File} = file:open(Path, [read]),
    try
        fold_lines(File, Fun, ok)
    after
        file:close(File)
    end.

fold_lines(File, Fun, Acc) ->
    case io:get_line(File, "") of
        eof    -> {ok, Acc};
        {error, R} -> {error, R};
        Line   ->
            case Fun(string:trim(Line), Acc) of
                {ok, NewAcc}    -> fold_lines(File, Fun, NewAcc);
                {stop, NewAcc}  -> {ok, NewAcc};
                {error, Reason} -> {error, Reason}
            end
    end.
```

### อ่านแบบ Buffered

```erlang
%% อ่านด้วย read_ahead สำหรับ performance
{ok, File} = file:open("large.txt", [read, read_ahead, binary]).

%% อ่าน N bytes ต่อครั้ง
read_chunks(File, ChunkSize) ->
    case file:read(File, ChunkSize) of
        {ok, Data}  -> {ok, Data};
        eof         -> eof;
        {error, R}  -> {error, R}
    end.
```

---

## 4. Writing Files

### เขียนทั้งไฟล์

```erlang
%% เขียน binary ทั้งหมดพร้อมกัน (atomic)
file:write_file("output.txt", <<"Hello, World!\n">>).

%% เขียน string
file:write_file("output.txt", "Hello, World!\n").

%% เขียน iolist
Content = ["Line 1\n", "Line 2\n", "Line 3\n"],
file:write_file("output.txt", Content).

%% append ต่อท้ายไฟล์
file:write_file("log.txt", "New log entry\n", [append]).
```

### เขียนทีละส่วน

```erlang
%% เปิดและเขียนทีละส่วน
write_records(Path, Records) ->
    {ok, File} = file:open(Path, [write, {encoding, utf8}]),
    try
        lists:foreach(fun(Record) ->
            io:format(File, "~p.~n", [Record])
        end, Records),
        ok
    after
        file:close(File)
    end.

%% เขียน formatted output
write_report(Path, Data) ->
    {ok, File} = file:open(Path, [write]),
    try
        io:format(File, "Report Generated: ~p~n~n", [erlang:timestamp()]),
        lists:foreach(fun({Name, Value}) ->
            io:format(File, "~-20s: ~p~n", [Name, Value])
        end, Data),
        ok
    after
        file:close(File)
    end.
```

### Delayed Write

```erlang
%% delayed_write: buffer writes สำหรับ performance
{ok, File} = file:open("output.txt", [write, delayed_write]).

%% ระบุ buffer size และ delay time
{ok, File} = file:open("output.txt", 
    [write, {delayed_write, 65536, 5000}]).
%% Buffer 64KB, flush ทุก 5 วินาที

%% sync: เขียน buffer ลง disk ทันที
file:sync(File).
```

---

## 5. File Metadata

```erlang
%% ข้อมูล file
{ok, Info} = file:read_file_info("test.txt").
%% #file_info{size=1024, type=regular, access=read_write,
%%             atime={...}, mtime={...}, ctime={...},
%%             mode=33188, links=1, major_device=8, ...}

%% Access struct fields
Size = Info#file_info.size.
Type = Info#file_info.type.     % regular, directory, symlink, etc.
Mode = Info#file_info.mode.
Mtime = Info#file_info.mtime.   % {{Y,M,D},{H,Min,S}}

%% ตรวจสอบ file ว่ามีอยู่
filelib:is_file("test.txt").
filelib:is_regular("test.txt").
filelib:is_dir("mydir").

%% ดู file size
filelib:file_size("test.txt").

%% Change file metadata
file:write_file_info("test.txt", 
    Info#file_info{mode = 8#755}).  % chmod 755

%% เวลาต่างๆ
{ok, Info2} = file:read_file_info("test.txt"),
ModTime = Info2#file_info.mtime,
io:format("Modified: ~p~n", [ModTime]).
```

---

## 6. Directory Operations

```erlang
%% สร้าง directory
file:make_dir("mydir").
filelib:ensure_dir("path/to/file.txt").  % สร้าง parent dirs ทั้งหมด

%% ลบ directory (ต้องว่าง)
file:del_dir("mydir").

%% รายการไฟล์ใน directory
{ok, Files} = file:list_dir(".").
%% ["file1.txt", "file2.txt", "subdir"]

%% ค้นหาด้วย wildcard
Files2 = filelib:wildcard("*.erl").
Files3 = filelib:wildcard("src/**/*.erl").

%% Walk directory recursively
walk_dir(Dir) ->
    {ok, Entries} = file:list_dir(Dir),
    lists:flatmap(fun(Entry) ->
        Path = filename:join(Dir, Entry),
        case filelib:is_dir(Path) of
            true  -> walk_dir(Path);
            false -> [Path]
        end
    end, Entries).

%% Copy file
file:copy("source.txt", "dest.txt").

%% Rename/Move
file:rename("old.txt", "new.txt").

%% Delete file
file:delete("temp.txt").

%% Get/Set current directory
{ok, CWD} = file:get_cwd().
file:set_cwd("/tmp").
```

### Directory Watching (ด้วย fs library)

```erlang
%% ใช้ fs library สำหรับ watch directory changes
%% rebar.config: {deps, [{fs, "..."}]}

start_watching(Dir) ->
    fs:start_link(my_watcher, Dir),
    fs:subscribe(my_watcher).

handle_events() ->
    receive
        {_Watcher, {event, Events}} ->
            lists:foreach(fun handle_event/1, Events),
            handle_events()
    end.

handle_event({Path, Event}) ->
    io:format("File ~s: ~p~n", [Path, Event]).
```

---

## 7. Random Access I/O

```erlang
%% เปิดไฟล์สำหรับ random access
{ok, File} = file:open("data.bin", [read, write, binary]).

%% ย้าย position
file:position(File, 0).          %% ไปต้นไฟล์
file:position(File, eof).        %% ไปท้ายไฟล์
file:position(File, 100).        %% ไป byte 100
file:position(File, {cur, 10}).  %% ไปข้างหน้า 10 bytes
file:position(File, {bof, 50}).  %% ไป 50 bytes จากต้น
file:position(File, {eof, -10}). %% ไป 10 bytes ก่อนท้ายไฟล์

%% อ่านที่ position ปัจจุบัน
{ok, Pos} = file:position(File, 50).
{ok, Data} = file:read(File, 10).

%% เขียนที่ position ปัจจุบัน
file:write(File, <<"new data">>).

%% ตัดไฟล์
file:truncate(File).  %% ตัดที่ current position
```

### Memory-Mapped Files (ด้วย Ports)

```erlang
%% Erlang ไม่มี mmap โดยตรง แต่ทำได้ผ่าน NIF หรือ Port driver
%% ใช้ ram_file module สำหรับ in-memory file operations

{ok, File} = file:open(Binary, [ram, binary, read, write]).
%% Binary คือ in-memory content
```

---

## 8. Streaming Large Files

```erlang
%% Process ไฟล์ขนาดใหญ่โดยไม่โหลดทั้งหมดเข้า memory

%% Approach 1: อ่านทีละ chunk
process_large_file(Path) ->
    {ok, File} = file:open(Path, [read, binary, read_ahead]),
    try
        process_chunks(File, <<>>, 0)
    after
        file:close(File)
    end.

process_chunks(File, Buffer, LineCount) ->
    case file:read(File, 65536) of  %% 64KB chunks
        {ok, Data} ->
            AllData = <<Buffer/binary, Data/binary>>,
            {Lines, Remaining} = split_lines(AllData),
            NewCount = LineCount + length(Lines),
            process_chunks(File, Remaining, NewCount);
        eof ->
            %% process remaining buffer
            {ok, LineCount};
        {error, Reason} ->
            {error, Reason}
    end.

split_lines(Bin) ->
    Parts = binary:split(Bin, <<"\n">>, [global]),
    case Parts of
        [] -> {[], <<>>};
        _ ->
            Last = lists:last(Parts),
            Lines = lists:droplast(Parts),
            {Lines, Last}
    end.

%% Approach 2: อ่านทีละบรรทัดด้วย read_ahead
stream_lines(Path) ->
    {ok, File} = file:open(Path, [read, read_ahead]),
    stream_lines_loop(File).

stream_lines_loop(File) ->
    case io:get_line(File, "") of
        eof -> done;
        {error, R} -> {error, R};
        Line ->
            process_line(string:trim(Line, trailing, "\n")),
            stream_lines_loop(File)
    end.

process_line(Line) ->
    io:format("~s~n", [Line]).
```

### CSV Processing

```erlang
%% Process CSV ขนาดใหญ่
process_csv(Path) ->
    {ok, File} = file:open(Path, [read, read_ahead]),
    try
        case io:get_line(File, "") of
            eof -> {ok, []};
            {error, R} -> {error, R};
            HeaderLine ->
                Headers = parse_csv_line(string:trim(HeaderLine, trailing, "\n")),
                process_csv_rows(File, Headers, 0)
        end
    after
        file:close(File)
    end.

process_csv_rows(File, Headers, Count) ->
    case io:get_line(File, "") of
        eof -> {ok, Count};
        {error, R} -> {error, R};
        Line ->
            Values = parse_csv_line(string:trim(Line, trailing, "\n")),
            Row = maps:from_list(lists:zip(Headers, Values)),
            process_row(Row),
            process_csv_rows(File, Headers, Count + 1)
    end.

parse_csv_line(Line) ->
    string:tokens(Line, ",").  %% simple CSV parser

process_row(Row) ->
    io:format("Row: ~p~n", [Row]).
```

---

## 9. Binary File I/O

### เขียนและอ่าน Binary Data

```erlang
%% เขียน binary data
write_binary_data(Path, Records) ->
    Data = encode_records(Records),
    file:write_file(Path, Data).

encode_records(Records) ->
    iolist_to_binary([encode_record(R) || R <- Records]).

encode_record({Name, Age, Score}) ->
    NameBin = list_to_binary(Name),
    NameLen = byte_size(NameBin),
    <<NameLen:16, NameBin/binary, Age:8, Score:32/float>>.

%% อ่าน binary data
read_binary_data(Path) ->
    case file:read_file(Path) of
        {ok, Bin} -> decode_records(Bin, []);
        Error     -> Error
    end.

decode_records(<<>>, Acc) ->
    {ok, lists:reverse(Acc)};
decode_records(<<NameLen:16, Name:NameLen/binary,
                 Age:8, Score:32/float, Rest/binary>>, Acc) ->
    Record = {binary_to_list(Name), Age, Score},
    decode_records(Rest, [Record|Acc]);
decode_records(_, _) ->
    {error, malformed_data}.
```

### term_to_binary / binary_to_term

```erlang
%% Serialize Erlang terms
serialize(Term) ->
    term_to_binary(Term).

deserialize(Binary) ->
    binary_to_term(Binary).

%% เก็บใน file
save_term(Path, Term) ->
    file:write_file(Path, term_to_binary(Term)).

load_term(Path) ->
    case file:read_file(Path) of
        {ok, Bin} ->
            try {ok, binary_to_term(Bin)}
            catch _:_ -> {error, invalid_data}
            end;
        Error -> Error
    end.

%% Compressed
save_compressed(Path, Term) ->
    file:write_file(Path, term_to_binary(Term, [compressed])).

%% Safe deserialize (ป้องกัน atom flooding)
safe_deserialize(Binary) ->
    try binary_to_term(Binary, [safe])
    catch _:_ -> {error, invalid_binary}
    end.
```

---

## 10. Path Operations

```erlang
%% filename module

%% Join paths
filename:join(["path", "to", "file.txt"]).
%% "path/to/file.txt"

filename:join("/absolute", "path/file.txt").
%% "/absolute/path/file.txt"

%% แยก path
filename:dirname("/path/to/file.txt").    % "/path/to"
filename:basename("/path/to/file.txt").   % "file.txt"
filename:basename("/path/to/file.txt", ".txt").  % "file"
filename:extension("/path/to/file.txt").  % ".txt"

%% Normalize path
filename:nativename("/path/to/../file").  % OS-specific

%% Absolute path
filename:absname("relative/path").   % ขึ้นกับ CWD

%% Split into components
filename:split("/path/to/file.txt").
%% ["/","path","to","file.txt"]

%% ตรวจสอบ
filelib:is_file("/path/to/file.txt").
filelib:is_dir("/path/to/dir").
filelib:is_regular("/path/to/file.txt").
filelib:is_symlink("/path/to/link").

%% Relative to Absolute
absolute(Path) ->
    case filename:pathtype(Path) of
        absolute -> Path;
        relative ->
            {ok, Cwd} = file:get_cwd(),
            filename:join(Cwd, Path)
    end.

%% Safe path join (ป้องกัน directory traversal)
safe_join(BaseDir, Path) ->
    AbsBase = filename:absname(BaseDir),
    AbsPath = filename:absname(filename:join(BaseDir, Path)),
    case string:prefix(AbsPath, AbsBase) of
        nomatch -> {error, path_traversal};
        _       -> {ok, AbsPath}
    end.
```

---

## 11. File Permissions

```erlang
%% Unix permissions ใน Erlang
%% Mode เป็น integer octal: 8#755 = 493 decimal

%% อ่าน permissions
{ok, Info} = file:read_file_info("test.txt"),
Mode = Info#file_info.mode,
io:format("Mode: ~8.8.0b (~.8B)~n", [Mode, Mode]).

%% แปลง mode เป็น readable string
mode_to_string(Mode) ->
    [
        case Mode band 16#100 of 16#100 -> $r; _ -> $- end,
        case Mode band 16#080 of 16#080 -> $w; _ -> $- end,
        case Mode band 16#040 of 16#040 -> $x; _ -> $- end,
        case Mode band 16#020 of 16#020 -> $r; _ -> $- end,
        case Mode band 16#010 of 16#010 -> $w; _ -> $- end,
        case Mode band 16#008 of 16#008 -> $x; _ -> $- end,
        case Mode band 16#004 of 16#004 -> $r; _ -> $- end,
        case Mode band 16#002 of 16#002 -> $w; _ -> $- end,
        case Mode band 16#001 of 16#001 -> $x; _ -> $- end
    ].

%% Change permissions
set_permissions(Path, Mode) ->
    {ok, Info} = file:read_file_info(Path),
    file:write_file_info(Path, Info#file_info{mode = Mode}).

%% Make executable
make_executable(Path) ->
    {ok, Info} = file:read_file_info(Path),
    NewMode = Info#file_info.mode bor 8#111,
    file:write_file_info(Path, Info#file_info{mode = NewMode}).
```

---

## 12. แบบฝึกหัด

### Exercise 1: File Processor

```erlang
%% สร้าง file_processor.erl ที่:
%% - อ่านไฟล์ text ทีละบรรทัด
%% - นับ: จำนวนบรรทัด, คำ, ตัวอักษร (เหมือน wc)
%% - ค้นหา pattern ใน file (grep-like)
%% - แทนที่ text (sed-like)

-module(file_processor).
-export([word_count/1, grep/2, replace/3]).

word_count(Path) ->
    case file:read_file(Path) of
        {ok, Bin} ->
            Content = binary_to_list(Bin),
            Lines = length(string:split(Content, "\n", all)),
            Words = length(string:tokens(Content, " \t\n")),
            Chars = length(Content),
            {ok, #{lines => Lines, words => Words, chars => Chars}};
        {error, Reason} ->
            {error, Reason}
    end.

grep(Pattern, Path) ->
    case re:compile(Pattern) of
        {ok, RE} ->
            {ok, File} = file:open(Path, [read]),
            try
                grep_lines(File, RE, 1, [])
            after
                file:close(File)
            end;
        {error, Err} ->
            {error, {invalid_pattern, Err}}
    end.

grep_lines(File, RE, LineNum, Matches) ->
    case io:get_line(File, "") of
        eof -> {ok, lists:reverse(Matches)};
        {error, R} -> {error, R};
        Line ->
            TrimmedLine = string:trim(Line, trailing, "\n"),
            case re:run(TrimmedLine, RE) of
                {match, _} ->
                    grep_lines(File, RE, LineNum+1,
                        [{LineNum, TrimmedLine}|Matches]);
                nomatch ->
                    grep_lines(File, RE, LineNum+1, Matches)
            end
    end.

replace(Pattern, Replacement, Path) ->
    case file:read_file(Path) of
        {ok, Bin} ->
            Content = binary_to_list(Bin),
            case re:replace(Content, Pattern, Replacement,
                           [global, {return, list}]) of
                Result when is_list(Result) ->
                    TmpPath = Path ++ ".tmp",
                    case file:write_file(TmpPath, Result) of
                        ok ->
                            file:rename(TmpPath, Path);
                        Error ->
                            file:delete(TmpPath),
                            Error
                    end;
                {error, Err} ->
                    {error, Err}
            end;
        {error, Reason} ->
            {error, Reason}
    end.
```

### Exercise 2: Config File Manager

```erlang
%% สร้าง config_file.erl ที่ manage config file
%% รองรับ format: key = value (INI-like)

-module(config_file).
-export([read/1, write/2, get/3, set/3]).

read(Path) ->
    case file:open(Path, [read]) of
        {ok, File} ->
            try parse_config(File, #{})
            after file:close(File)
            end;
        {error, enoent} ->
            {ok, #{}};
        {error, Reason} ->
            {error, Reason}
    end.

parse_config(File, Config) ->
    case io:get_line(File, "") of
        eof -> {ok, Config};
        {error, R} -> {error, R};
        Line ->
            TrimLine = string:trim(Line),
            case parse_line(TrimLine) of
                skip ->
                    parse_config(File, Config);
                {key, K, V} ->
                    parse_config(File, Config#{K => V})
            end
    end.

parse_line("") -> skip;
parse_line("#" ++ _) -> skip;  %% comment
parse_line(Line) ->
    case string:split(Line, "=") of
        [Key, Value] ->
            K = list_to_atom(string:trim(Key)),
            V = string:trim(Value),
            {key, K, V};
        _ -> skip
    end.

write(Path, Config) ->
    Lines = maps:fold(fun(K, V, Acc) ->
        [io_lib:format("~s = ~s~n", [K, V]) | Acc]
    end, [], Config),
    file:write_file(Path, Lines).

get(Path, Key, Default) ->
    case read(Path) of
        {ok, Config} -> maps:get(Key, Config, Default);
        {error, _}   -> Default
    end.

set(Path, Key, Value) ->
    case read(Path) of
        {ok, Config} ->
            write(Path, Config#{Key => Value});
        Error ->
            Error
    end.
```

### Exercise 3: Log File Analyzer

```erlang
%% สร้าง log_analyzer.erl ที่วิเคราะห์ log files
%% Format: [YYYY-MM-DD HH:MM:SS] [LEVEL] Message

-module(log_analyzer).
-export([analyze/1, filter_by_level/2, count_by_level/1]).

-record(log_entry, {
    timestamp :: calendar:datetime(),
    level     :: error | warn | info | debug,
    message   :: binary()
}).

analyze(Path) ->
    {ok, File} = file:open(Path, [read, binary, read_ahead]),
    try
        collect_entries(File, [])
    after
        file:close(File)
    end.

collect_entries(File, Acc) ->
    case io:get_line(File, "") of
        eof -> {ok, lists:reverse(Acc)};
        {error, R} -> {error, R};
        Line ->
            TrimLine = string:trim(iolist_to_binary(Line), trailing, "\n"),
            case parse_log_line(TrimLine) of
                {ok, Entry} -> collect_entries(File, [Entry|Acc]);
                skip        -> collect_entries(File, Acc)
            end
    end.

parse_log_line(Line) ->
    Pattern = "\\[(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2})\\] \\[(\\w+)\\] (.+)",
    case re:run(Line, Pattern, [{capture, [1,2,3], binary}]) of
        {match, [TS, Level, Msg]} ->
            {ok, #log_entry{
                timestamp = parse_timestamp(TS),
                level     = parse_level(Level),
                message   = Msg
            }};
        nomatch ->
            skip
    end.

parse_timestamp(TS) ->
    %% parse "2024-01-15 12:30:00"
    [Date, Time] = binary:split(TS, <<" ">>),
    [Y,M,D] = [binary_to_integer(X) || X <- binary:split(Date, <<"-">>, [global])],
    [H,Min,S] = [binary_to_integer(X) || X <- binary:split(Time, <<":">>, [global])],
    {{Y,M,D},{H,Min,S}}.

parse_level(<<"ERROR">>) -> error;
parse_level(<<"WARN">>)  -> warn;
parse_level(<<"INFO">>)  -> info;
parse_level(<<"DEBUG">>) -> debug;
parse_level(_)           -> unknown.

filter_by_level(Entries, Level) ->
    [E || E <- Entries, E#log_entry.level =:= Level].

count_by_level(Entries) ->
    lists:foldl(fun(#log_entry{level = L}, Acc) ->
        maps:update_with(L, fun(V) -> V+1 end, 1, Acc)
    end, #{}, Entries).
```

---

## สรุป Part 09

ใน Part นี้คุณได้เรียนรู้:

✅ Standard I/O — io:format, io:get_line  
✅ File open/close — modes, error handling  
✅ Reading files — ทั้งไฟล์, ทีละบรรทัด, buffered  
✅ Writing files — ทั้งไฟล์, ทีละส่วน, delayed write  
✅ File metadata — file_info  
✅ Directory operations  
✅ Random access I/O  
✅ Streaming large files  
✅ Binary file I/O  
✅ Path operations ด้วย filename module  
✅ File permissions  

---

## ต่อไป: [Part 10 — Error Handling พื้นฐาน](../part10/README.md)

ใน Part ถัดไปเราจะเรียนรู้:
- Error handling philosophy ใน Erlang
- "Let it crash" principle
- try/catch ขั้นสูง
- Error types และ categorization
- Defensive programming
- Error logging

---

*Part 09/100 | [← ก่อนหน้า](../part08/README.md) | [ถัดไป →](../part10/README.md)*
