みなさん、こんにちは！ 2週間ぶりのご無沙汰、MUGENUP の osada です。

`ruby 2.1.0`、`Rails 4.1beta` で開発した、`みなゲー編集部`が正式リリースとなりました！
よろしくお願いします。
[みなゲー編集部](https://edit.mina-game.com/users/login)

さて、そんな`Rails 4.1`の新機能として、プレローダー`spring`が標準装備となりました。
本日は`spring`と`paralell_tests`を併用する方法についてのお話です。

要旨は下記となります。

> PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" を設定して、`paralell_tests`に`spring`を使わせない
> `bin/rspec`を書き換えて、始めのプロセスだけ`spring`を使う
> `FailuresLogger` を使って、失敗したテストを再実行する
> `RuntimeLogger`を使って、テストのグループを平均化する

では、よろしくお願いします。

## PARALLEL_TESTS_EXECUTABLE で spring を回避する

`Rails 4.1`において、`rspec`では成功するのに、`paralell_tests`ではテストの多くが失敗する、という現象に遭いました。
どうやら、`spring`が原因だろう、ということです。

通常、`rake`や`rspec`コマンドを使うとき、コードを読み込んで、メモリ上に`Rails`の`app`インスタンスを生成します。
この処理はかなり重い処理なので、毎回作るのではなく、一度作ったアプリを使いまわそう、というのが`spring`の動きです。

この`spring`は`bin_stub`として提供され、`bin/`というディレクトリに
`rails`, `rspec`, `rake`のコマンドが用意されます。

```
be bin/rake db:create
```

のように、通常のコマンドの代わりに、`bin/`下のコマンドを呼び出すことで、`spring`が使用されます。


一方、`paralell_tests`というのは、`rake`などの処理を複数のプロセスに分けて同時に処理する方法です。

例えば、`parallel:spec`を実行すると、下記の4つのコマンドが実行されます。

> ※`be` は `bundle exec` の `alias` です

```
$ be rake parallel:spec

TEST_ENV_NUMBER=;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rspec
TEST_ENV_NUMBER=3;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rspec
TEST_ENV_NUMBER=4;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rspec
TEST_ENV_NUMBER=2;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rspec
```

しかし、これに`spring`を使用すると、4つのプロセスがあるのに、実際に稼働する`app`は1つになってしまいます。


よって、テストが失敗してしまうのです。

`paralell_tests`は`bin/rspec`があると、それを使用しようとします。これをどうにかしましょう。
```
$ be rake parallel:spec

TEST_ENV_NUMBER=;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;bin/rspec
...
```

結論からいうと、`PARALLEL_TESTS_EXECUTABLE`を設定することで、`bin/rspec`の使用を回避できます。

```
PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" be rake parallel:spec

TEST_ENV_NUMBER=;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;bundle exec rspec
...
```

この理由は、`parallel_tests`は、実行するファイルを下記のように設定しているからです。

```
def executable
  ENV['PARALLEL_TESTS_EXECUTABLE'] || determine_executable
end
```

また`rspec`の`determine_executable`は下記であり、[`bin/rspec`, `script/spec`, `bundle exec rspec`, `spec`, `rspec`] のどれかを起動する事がわかります。

```rb
def determine_executable
  cmd = case
  when File.exists?("bin/rspec")
    "bin/rspec"
  when File.file?("script/spec")
    "script/spec"
  when ParallelTests.bundler_enabled?
    cmd = (run("bundle show rspec-core") =~ %r{Could not find gem.*} ? "spec" : "rspec")
    "bundle exec #{cmd}"
  else
    %w[spec rspec].detect{|cmd| system "#{cmd} --version > /dev/null 2>&1" }
  end

  cmd or raise("Can't find executables rspec or spec")
end
```

以上、本項の結論は、
> `parallel_tests`に、`spring`を使わせないためには、`PARALLEL_TESTS_EXECUTABLE`を設定する
となります。


## `bin/rspec`を書き換えて、始めのプロセスだけ`spring`を使う

しかしせっかくの`spring`です。
4つの内1つだけで`spring`を使い、
残り3つをを別のプロセスで動貸せば良いのではないでしょうか？

相手は`bin_stub`ですから、修正はとても簡単です。

`bin/rspec`のこれを、

```
if !Process.respond_to?(:fork) || Gem::Specification.find_all_by_name("spring").empty?
  exec "bundle", "exec", "rspec", *ARGV
else
  ARGV.unshift "rspec"
  load Gem.bin_path("spring", "spring")
end
```

こうしましょう！

```
if !Process.respond_to?(:fork) || Gem::Specification.find_all_by_name("spring").empty?
  exec "bundle", "exec", "rspec", *ARGV
else
  if ENV["TEST_ENV_NUMBER"].nil? || ENV["TEST_ENV_NUMBER"].empty?
    ARGV.unshift "rspec"
    load Gem.bin_path("spring", "spring")
  else
    exec "bundle", "exec", "rspec", *ARGV
  end
end
```

`ENV["TEST_ENV_NUMBER"].nil?` は通常の`bin/rspec`起動時です。
`ENV["TEST_ENV_NUMBER"].empty?`は、`parallel_tests`の一番始めのプロセスの時です。

つまり、

> `bin/rspec`として起動したとき、または、`parallel_tests`の一番始めのプロセスの時に、`spring`を使用する。

という`bin_stub`になりました。

さて、肝心のスピードアップですが、`feature`テストなどが多いため、私のプロジェクトでは違いが現れませんでした。
参考までに`spec/model`に適用した結果を載せておきます。

```
[feature/parallel stash]~/projects/mugenup/workstation: time RAILS_ENV=test PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" bundle exec rake parallel:spec[model]
...
444 examples, 0 failures

Took 216.680576 seconds

real    3m47.661s
user    3m20.585s
sys     0m25.249s
```


```
[feature/parallel stash]~/projects/mugenup/workstation: time RAILS_ENV=test bundle exec bin/rake parallel:spec[model]

...
444 examples, 0 failures

Took 212.844729 seconds

real    3m37.395s
user    0m1.378s
sys     0m0.285s
```

`10 sec`ほど速くなっているように見えますが、誤差の範囲かもしれません。
もし「*速くなったよ！*」という方がいらっしゃいましたら、ご一報ください。
私が嬉しいです。

## `FailuresLogger` を使って、失敗したテストを再実行する

`feature`テストは実際のブラウザの挙動を再現するため、
タイムアウトするなどの理由により、
タイミングによっては失敗することもあります。
`parallel_tests`を使うと、特に落ちやすくなります。

たまたま落ちたのか、本当に失敗しているのかを確認するため、
落ちたテストのみを再度実行するのですが、
その時に役立つのが、`FailuresLogger`です。

```
# .rspec_parallel

--format progress
--format ParallelTests::RSpec::FailuresLogger --out tmp/failing_specs.log
```

これには失敗したテストがログに残ります。



ここから、再度テストを実行しましょう。

```
$ grep rspec tmp/failing_specs.log | awk '{print $2}' | xargs bundle exec rspec

/Users/osada/projects/mugenup/workstation/vendor/bundle/gems/rspec-core-2.14.7/lib/rspec/core/configuration.rb:896:in `load': cannot load such file -- /Users/osada/projects/mugenup/workstation/spec/views/projects/edit.html.erb_spec.rb:34 (LoadError)
```

あら、失敗してしまいました。なぜでしょうか？
実際にファイルを開いてみたところ、`ascii color code`が含まれていました。

```
<1b>[31mrspec ./spec/views/projects/edit.html.erb_spec.rb:34<1b>[0m <1b>[36m# projects/edit renders _form partial<1b>[0m
<1b>[31mrspec ./spec/features/post_spec.rb:66<1b>[0m <1b>[36m# Post after_createでwork_flowのstatusが変更される normal user change status<1b>[0m
```

たしかに、これでは、`rspec`が通りませんね。
といっても`--color`を消してしまうのも、嬉しくありません。

そこで、正規表現を使って、抜き出すことにしました。
- rspec から始まり、「ドット、スラッシュ、コロン、英数字とアンダースコア」で構成される文字列を抽出する

```
$ ruby -ne 'puts $1 if /(?<=rspec )([\.\/:\w]+)/' tmp/failing_specs.log | xargs bundle exec rspec
```

これで、失敗したテストのみ、再実行することができます。
`ruby`でワンライナーを書くのは、少し大変ですね。


## `RuntimeLogger`を使って、テストのグループを平均化する

さて、`paralell_tests`はデフォルトでは、ファイルサイズ順に均等化します。
大きい順に並べて、4つのグループに順番に追加していくわけです。

`RuntimeLogger`を使うことで、テストにかかった時間で、均等化することができます。

```
--format progress
--format ParallelTests::RSpec::RuntimeLogger --out tmp/parallel_runtime_rspec.log
```


`tmp/parallel_runtime_rspec.log`には、テストファイル名と、その実行時間が記録されています。

```
[feature/parallel stash]~/projects/mugenup/workstation: tail tmp/parallel_runtime_rspec.log
./spec/views/sub_projects/show.html.erb_spec.rb:2.261982
./spec/views/projects/show.html.erb_spec.rb:2.806851
......
```

テストのグループ分けのとき、`with_runtime_info`メソッドが呼ばれます。
`runtime_log`つまり`tmp/parallel_runtime_rspec`があるとき、かつ、
今回の`tests`ファイル群が、このログの中に含まれていると考えられるときに、
使用します。
`puts "Using recorded test runtime"`が出力されれば、成功です。

```rb
def with_runtime_info(tests)
  lines = File.read(runtime_log).split("\n") rescue []

  # use recorded test runtime if we got enough data
  if lines.size * 1.5 > tests.size
    puts "Using recorded test runtime"
    times = Hash.new(1)
    lines.each do |line|
      test, time = line.split(":")
      next unless test and time
      times[File.expand_path(test)] = time.to_f
    end
    tests.sort.map{|test| [test, times[File.expand_path(test)]] }
  else # use file sizes
    tests.sort.map{|test| [test, File.stat(test).size] }
  end
end
```

実際に、使用したときと、使用していないときを比較してみます。

```
[feature/parallel stash]~/projects/mugenup/workstation: RAILS_ENV=test bundle exec bin/rake parallel:spec[model]

Finished in 2 minutes 19.6 seconds
78 examples, 0 failures

Finished in 3 minutes 19.1 seconds
124 examples, 0 failures

Finished in 3 minutes 10.7 seconds
116 examples, 0 failures

Finished in 3 minutes 13.1 seconds
126 examples, 0 failures
```

使用していないときの処理時間は`2:19.6 〜 3:19.1`であり、`60`秒以上のズレがあります。

一方、`runtime_log`を使用したときは下記です。
`Using recorded test runtime`が表示されていることが確認できます。

```
[feature/parallel stash]~/projects/mugenup/workstation: RAILS_ENV=test bundle exec bin/rake parallel:spec[model]
Using recorded test runtime

Finished in 3 minutes 7.8 seconds
102 examples, 0 failures

Finished in 2 minutes 58.3 seconds
108 examples, 0 failures

Finished in 3 minutes 10.2 seconds
123 examples, 0 failures

Finished in 3 minutes 11.8 seconds
111 examples, 0 failures
```

`2:58.3 〜 3:11.8`であり、`13`秒ほどのズレでした。

残念ながら、こちらも大きなスピードアップは感じられませんでしたが、
速くなったよ、という方はご一報いただけると嬉しいです。


## まとめ

`parallel_tests`と`spring`の使い方、いかがだったでしょうか？

> PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" を設定して、`paralell_tests`に`spring`を使わせない
> `bin/rspec`を書き換えて、始めのプロセスだけ`spring`を使う
> `FailuresLogger` を使って、失敗したテストを再実行する
> `RuntimeLogger`を使って、テストのグループを平均化する

弊社でも**テストが遅い**ということが、問題になっており、日々改善に勤しんでおります。
*こんなやり方あるよ！*という情報をお持ちの方、いらっしゃいましたら、
ご教授いただけますと幸いです。
よろしくお願いします！




```
$ PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" time bundle exec rake parallel:spec RAILS_ENV=test

1720 examples, 2 failures

Took 727.141463 seconds
Tests Failed
      737.74 real       712.82 user        53.33 sys
```
```
1722 examples, 8 failures, 2 pendings

Took 772.554888 seconds
Tests Failed
      783.38 real       706.36 user        54.55 sys
```

```
--format progress
--format ParallelTests::RSpec::RuntimeLogger --out tmp/parallel_runtime_rspec.log
--format ParallelTests::RSpec::FailuresLogger --out tmp/failing_specs.log
```

```rb
def with_runtime_info(tests)
  lines = File.read(runtime_log).split("\n") rescue []

  # use recorded test runtime if we got enough data
  if lines.size * 1.5 > tests.size
    puts "Using recorded test runtime"
    times = Hash.new(1)
    lines.each do |line|
      test, time = line.split(":")
      next unless test and time
      times[File.expand_path(test)] = time.to_f
    end
    tests.sort.map{|test| [test, times[File.expand_path(test)]] }
  else # use file sizes
    tests.sort.map{|test| [test, File.stat(test).size] }
  end
end
```

```
1720 examples, 2 failures

Took 739.284838 seconds
Tests Failed
      749.92 real       709.67 user        54.02 sys
```

```
[feature/parallel stash]~/projects/mugenup/workstation: tail tmp/parallel_runtime_rspec.log
./spec/models/concerns/users/search_spec.rb:8.332869
./spec/views/sub_projects/show.html.erb_spec.rb:2.261982
./spec/routing/offers_routing_spec.rb:2.644081
./spec/views/offer_notifications/index.html.erb_spec.rb:1.305509
./spec/views/projects/show.html.erb_spec.rb:2.806851
./spec/views/work_flow_types/index.html.erb_spec.rb:0.608363
./spec/features/bills_spec.rb:6.471469
```

```
grep rspec tmp/failing_specs.log | awk '{print $2}' | xargs bundle exec rspec
```

失敗。

```
/Users/osada/projects/mugenup/workstation/vendor/bundle/gems/rspec-core-2.14.7/lib/rspec/core/configuration.rb:896:in `load': cannot load such file -- /Users/osada/projects/mugenup/workstation/spec/views/projects/edit.html.erb_spec.rb:34 (LoadError)
```


ascii color code が含まれていた。

```
<1b>[31mrspec ./spec/views/projects/edit.html.erb_spec.rb:34<1b>[0m <1b>[36m# projects/edit renders _form partial<1b>[0m
<1b>[31mrspec ./spec/features/post_spec.rb:66<1b>[0m <1b>[36m# Post after_createでwork_flowのstatusが変更される normal user change status<1b>[0m
```

正規表現を使って、抜き出す。
- rspec から始まり、「ドット、スラッシュ、コロン、英数字とアンダースコア」で構成される文字列を抽出する

```
ruby -ne 'puts $1 if /(?<=rspec )([\.\/:\w]+)/' tmp/failing_specs.log | xargs bundle exec rspec
```

```
if !Process.respond_to?(:fork) || Gem::Specification.find_all_by_name("spring").empty?
  exec "bundle", "exec", "rspec", *ARGV
else
  ARGV.unshift "rspec"
  load Gem.bin_path("spring", "spring")
end
```


```
if !Process.respond_to?(:fork) || Gem::Specification.find_all_by_name("spring").empty?
  exec "bundle", "exec", "rspec", *ARGV
else
  if ENV["TEST_ENV_NUMBER"].nil? || ENV["TEST_ENV_NUMBER"].empty?
    ARGV.unshift "rspec"
    load Gem.bin_path("spring", "spring")
  else
    exec "bundle", "exec", "rspec", *ARGV
  end
end
```

```
1720 examples, 1 failure

Took 767.907344 seconds
Tests Failed

real    12m58.882s
user    8m36.441s
sys     0m43.041s
```


1720 examples, 3 failures

Took 750.735822 seconds
Tests Failed

real    12m40.152s
user    8m39.240s
sys     0m42.535s


テスト件数が多いため、`[grosser/parallel_tests](https://github.com/grosser/parallel_tests)` を使用しています。
しかし、`parallel_tests`のときには失敗し、`rspec`だと成功する、ということがあり、悩ましく思っていました。
そこで、`parallel_test`がどのように動いているか、調べてみました。

## parallel:create

まずは`parallel:create`から調べます。

```ruby
# 2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests/tasks.rb

namespace :parallel do
  desc "create test databases via db:create --> parallel:create[num_cpus]"
  task :create, :count do |t,args|
    ParallelTests::Tasks.run_in_parallel("rake db:create RAILS_ENV=#{ParallelTests::Tasks.rails_env}", args)
  end
```

`run_in_parallel`に対して、お馴染みの`rake db:create`を発行しています。
`rails_env`は`ENV['RAILS_ENV'] || 'test'`が入っているので、`run_in_parallel`を確認します。

```ruby
# 2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests/tasks.rb

def run_in_parallel(cmd, options={})
  count = " -n #{options[:count]}" unless options[:count].to_s.empty?
  executable = File.expand_path("../../../bin/parallel_test", __FILE__)
  command = "#{executable} --exec '#{cmd}'#{count}#{' --non-parallel' if options[:non_parallel]}"
  abort unless system(command)
end
```

1. `count = " -n #{options[:count]}" unless options[:count].to_s.empty?`

`options[:count]`というのは、タスク起動時にオプションを渡していたら、ということです。

例えば
`bundle exec rake parallel:create[3]` と書けば、
`task :create, :count do |t,args|` の中で、`args`にハッシュとして渡ります。

`puts args #=> {:count=>"3"}`

今回は、何も渡さないため、この行は無視します

2. `executable = File.expand_path("../../../bin/parallel_test", __FILE__)`

このファイルから、3階層上まで上り、`bin/parallel_test` ファイルを実行ファイルとします。
ファイル内容は下記です。

```ruby
# 2.1.0/gems/parallel_tests-0.16.9/bin/parallel_test

#!/usr/bin/env ruby
require 'rubygems' if RUBY_VERSION < "1.9"
$LOAD_PATH << File.expand_path("../../lib", __FILE__)
require "parallel_tests"

ParallelTests::CLI.new.run(["--type", "test"] + ARGV)
```

要点は最下行の`ParallelTests::CLI.new.run(["--type", "test"] + ARGV)` です。
`ParallelTests`は`CLI`で起動している、つまり、**コマンドプロンプトから起動する**ということが分かります。

3. コマンドを作成して、起動

```ruby
command = "#{executable} --exec '#{cmd}'#{count}#{' --non-parallel' if options[:non_parallel]}"
abort unless system(command)
```

先に下の行から確認します。`system`は外部コマンドを実行します。正常終了で`true`が返りますので、異常終了で、`abort`します

では、上の行、`command`の確認です。これは、実行結果を`puts`した方が早いので御覧ください。

```
$ bundle exec rake parallel:create

/Users/osada/projects/mugenup/hoge/vendor/bundle/ruby/2.1.0/gems/parallel_tests-0.16.9/bin/parallel_test --exec 'rake db:create RAILS_ENV=test'

$ bundle exec rake parallel:create[3]

/Users/osada/projects/mugenup/hoge/vendor/bundle/ruby/2.1.0/gems/parallel_tests-0.16.9/bin/parallel_test --exec 'rake db:create RAILS_ENV=test' -n 3
```

`parallel:create`は、`CLI`を起動するラッパーだということがわかりました。
ではその`ParallelTests::CLI`を確認します。

## ParallelTests::CLI

これは`2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests/cli.rb` にあります。

```ruby
def run(argv)
  options = parse_options!(argv)

  num_processes = ParallelTests.determine_number_of_processes(options[:count])
  num_processes = num_processes * (options[:multiply] || 1)

  if options[:execute]
    execute_shell_command_in_parallel(options[:execute], num_processes, options)
  else
    run_tests_in_parallel(num_processes, options)
  end
end
```

さて、ここも確認していきたいのですが、`parse_options`が**72行あります!**。
さすがに大変なので説明は省略します。
詳細は[grosser/parallel_tests](https://github.com/grosser/parallel_tests)のオプションをご参考ください。

1. num_processes = ParallelTests.determine_number_of_processes(options[:count])

使用するプロセス数の決定です。

```
# 2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests.rb

def determine_number_of_processes(count)
  [
    count,
    ENV["PARALLEL_TEST_PROCESSORS"],
    Parallel.processor_count
  ].detect{|c| not c.to_s.strip.empty? }.to_i
end
```

`count`か`ENV["PARALLEL_TEST_PROCESSORS"]`か`Parallel.processor_count`で定義された数を返します。

ほとんどの場合、`Parallel.processor_count`を使用するのではないでしょうか。

[grosser/parallel](https://github.com/grosser/parallel)というのは、並列実行用のgemです。
マルチプロセスにも、マルチスレッドにも対応しているようです。
スピードが必要な場合に、使ってみたいものです。

> Run any code in parallel Processes(> use all CPUs) or Threads(> speedup blocking operations).
> Best suited for map-reduce or e.g. parallel downloads/uploads.


```ruby
[1] pry(main)> require "parallel"
=> false
[2] pry(main)> Parallel.processor_count
=> 4
```

続けます。

2. `num_processes = num_processes * (options[:multiply] || 1)`

これはオプションにあります。が、どんなときに使うのでしょうか？
```
-m, --multiply-processes [FLOAT] use given number as a multiplier of processes to run
```

3. 並列実行する

```
if options[:execute]
  execute_shell_command_in_parallel(options[:execute], num_processes, options)
else
  run_tests_in_parallel(num_processes, options)
end
```

シェルコマンドの実行と、テストの実行の2種類に分かれます。

今回は`--exec 'rake db:create RAILS_ENV=test'`という引数でしたので、シェルコマンドの実行です。
`execute_shell_command_in_parallel(options[:execute], num_processes, options)`をみていきます。

```
def execute_shell_command_in_parallel(command, num_processes, options)
  runs = (0...num_processes).to_a
  results = if options[:non_parallel]
    runs.map do |i|
      ParallelTests::Test::Runner.execute_command(command, i, num_processes, options)
    end
  else
    execute_in_parallel(runs, num_processes, options) do |i|
      ParallelTests::Test::Runner.execute_command(command, i, num_processes, options)
    end
  end.flatten

  abort if results.any? { |r| r[:exit_status] != 0 }
end
```

このgemで`non_parallel`にどのような利用法があるのかわかりませんが、今回は`else`節を見ていきます。

ここが並列実行の心臓部のようです。

### `execute_in_parallel`

```
def execute_in_parallel(items, num_processes, options)
  Tempfile.open 'parallel_tests-lock' do |lock|
    return Parallel.map(items, :in_threads => num_processes) do |item|
      result = yield(item)
      report_output(result, lock) if options[:serialize_stdout]
      result
    end
  end
end
```

`Tempfile.open 'parallel_tests-lock' do |lock|` は、
`report_output(result, lock) if options[:serialize_stdout]`用であり、
全ての実行結果が揃ってから出力するために、他の実行をロックするものです。
今回は解説をスキップします。

今回、`items`に入るのは、`[0, 1, 2, 3]`。
`num_processes`には`4`が入りますので、`in_threads => num_processes` は、**4つのマルチスレッドで動く**を意味します。

`yield`にスレッド番号を渡しながら、並列実行する、というイメージでしょうか。

`yeild`に入るのは、

```ruby
execute_in_parallel(runs, num_processes, options) do |i|
  ParallelTests::Test::Runner.execute_command(command, i, num_processes, options)
end
```


```ruby
# 2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests/test/runner.rb

def execute_command(cmd, process_number,  num_processes, options)
  env = (options[:env] || {}).merge(
    "TEST_ENV_NUMBER" => test_env_number(process_number),
    "PARALLEL_TEST_GROUPS" => num_processes
  )
  cmd = "nice #{cmd}" if options[:nice]
  execute_command_and_capture_output(env, cmd, options[:serialize_stdout])
end
```

`execute_command_and_capture_output` は、
色々やっているように見えますが、環境変数を整形してパイプを開いているだけです。


```
def execute_command_and_capture_output(env, cmd, silence)
  # make processes descriptive / visible in ps -ef
  windows = RbConfig::CONFIG['host_os'] =~ /cygwin|mswin|mingw|bccwin|wince|emx/
  separator = windows ? ' & ' : ';'
  exports = env.map do |k,v|
    if windows
      "(SET \"#{k}=#{v}\")"
    else
      "#{k}=#{v};export #{k}"
    end
  end.join(separator)
  cmd = "#{exports}#{separator}#{cmd}"

  output = open("|#{cmd}", "r") { |output| capture_output(output, silence) }
  exitstatus = $?.exitstatus

  {:stdout => output, :exit_status => exitstatus}
end
```

さて、`cmd`を`puts`してみます。


```
$ be rake parallel:create

TEST_ENV_NUMBER=;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rake db:create RAILS_ENV=test
TEST_ENV_NUMBER=3;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rake db:create RAILS_ENV=test
TEST_ENV_NUMBER=4;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rake db:create RAILS_ENV=test
TEST_ENV_NUMBER=2;export TEST_ENV_NUMBER;PARALLEL_TEST_GROUPS=4;export PARALLEL_TEST_GROUPS;rake db:create RAILS_ENV=test
```




今回は、何も渡さないため、実行されるのは`else`節、つまり、
`run_tests_in_parallel(num_processes, options)`です。


## 並列実行

かなりコアに近づいてきました。ここからは、さらに詳細に見ていきます。
`run_tests_in_parallel(num_processes, options)`です。

```
def run_tests_in_parallel(num_processes, options)
  test_results = nil

  report_time_taken do
    groups = @runner.tests_in_groups(options[:files], num_processes, options)

    test_results = if options[:only_group]
      group_to_run = groups[options[:only_group] - 1]

      [run_tests(group_to_run, 0, num_processes, options)]
    else
      report_number_of_tests(groups)

      execute_in_parallel(groups, groups.size, options) do |group|
        run_tests(group, groups.index(group), num_processes, options)
      end
    end

    report_results(test_results)
  end

  abort final_fail_message if any_test_failed?(test_results)
end
```

1. `report_time_taken`

```
def report_time_taken
  start = Time.now
  yield
  puts "\nTook #{Time.now - start} seconds"
end
```
実行した時間を計測するためにブロックを使っています。
これだけでも覚えておくと、他に活かせそうですね。


2. `groups = @runner.tests_in_groups(options[:files], num_processes, options)`
ここではテスト対象のファイル群を、プロセスごとに割り当てるために、グループ化する処理を行います。しかし今回は対象のファイル群がないので、後述のテストの項目で説明することとし、ここではスキップします。
`groups`は空リストです。

3. `execute_in_parallel`

`only-group`が指定されていると、そのグループだけを実行するようですが、今回は、`else`節を見ていきます。

```
--only-group [INTEGER]       Group the files, but only run the group specified here. Requires group-by filesize (will be set automatically if group-by is blank and only-group is specified)
```

```
test_results = if options[:only_group]
  group_to_run = groups[options[:only_group] - 1]

  [run_tests(group_to_run, 0, num_processes, options)]
else
  report_number_of_tests(groups)

  execute_in_parallel(groups, groups.size, options) do |group|
    run_tests(group, groups.index(group), num_processes, options)
  end
end
```




### rspec のときの動き


ここで、グループ化をしているようです。
グループ化というのは、テスト実行時にどのファイルを

`@runner`がいきなり出てきました。これは何でしょうか？
と思ったら、実はスキップした`parse_options`で定義されています。

```
@runner = load_runner(type)
```
```
def load_runner(type)
  require "parallel_tests/#{type}/runner"
  klass_name = "ParallelTests::#{type.capitalize.sub("Rspec", "RSpec")}::Runner"
  klass_name.split('::').inject(Object) { |x, y| x.const_get(y) }
end
```

`type`に入るのは、`CLI`起動時に渡されるものです。
例えば、`rspec`であれば、この`type`には`rspec`が入ります。

```ruby
# 2.1.0/gems/parallel_tests-0.16.9/bin/parallel_test

ParallelTests::CLI.new.run(["--type", "test"] + ARGV)
```

```ruby
# 2.1.0/gems/parallel_tests-0.16.9/bin/parallel_rspec

ParallelTests::CLI.new.run(["--type", "rspec"] + ARGV)
```

動的に`require`することで、それぞれの`type`の`runner`を読み込みます。
```
|-- cucumber
|   |-- failures_logger.rb
|   |-- runner.rb
|   |-- scenario_line_logger.rb
|   |-- scenarios.rb
|-- gherkin
|   |-- io.rb
|   |-- listener.rb
|   |-- runner.rb
|   |-- runtime_logger.rb
|-- rspec
|   |-- failures_logger.rb
|   |-- logger_base.rb
|   |-- runner.rb
|   |-- runtime_logger.rb
|   |-- summary_logger.rb
|-- spinach
|   |-- runner.rb
|-- test
|   |-- runner.rb
|   |-- runtime_logger.rb
```

では改めて、`groups = @runner.tests_in_groups(options[:files], num_processes, options)`を確認します。

今回は、`test/runner.rb` の、`test_in_groups`です。

```ruby
# finds all tests and partitions them into groups
def tests_in_groups(tests, num_groups, options={})
  tests = find_tests(tests, options)

  tests = if options[:group_by] == :found
    tests.map { |t| [t, 1] }
  elsif options[:group_by] == :filesize
    with_filesize_info(tests)
  else
    with_runtime_info(tests)
  end
  Grouper.in_even_groups_by_size(tests, num_groups, options)
end
```

`find_tests`は、テスト対象のファイルリストを返すメソッドです。
今回はテストではないので、空リストが返ります。

`options[:group_by]`は、ファイルリストをグループ化する方法の選択です。
返り値がないことから、破壊型のソーティングを行うことが読み取れますが、
今回は`tests`が空ですので、詳細は後述とします。


`find_tests`

```ruby
def find_tests(tests, options = {})
  (tests || []).map do |file_or_folder|
    if File.directory?(file_or_folder)
      files = files_in_folder(file_or_folder, options)
      files.grep(test_suffix).grep(options[:pattern]||//)
    else
      file_or_folder
    end
  end.flatten.uniq
end
```



```
# 2.1.0/gems/parallel_tests-0.16.9/lib/parallel_tests/grouper.rb

def in_even_groups_by_size(items, num_groups, options= {})
  groups = Array.new(num_groups) { {:items => [], :size => 0} }

  # add all files that should run in a single process to one group
  (options[:single_process] || []).each do |pattern|
    matched, items = items.partition { |item, size| item =~ pattern }
    matched.each { |item, size| add_to_group(groups.first, item, size) }
  end

  groups_to_fill = (options[:isolate] ? groups[1..-1] : groups)
  group_features_by_size(items_to_group(items), groups_to_fill)

  groups.map!{|g| g[:items].sort }
end
```


```
> -s, --single [PATTERN]           Run all matching files in the same process
> -i, --isolate                    Do not run any other tests in the group used by --single(-s)
```

`--single`で入力したパターンを同じグループにする。
そして、`--isolate`を指定すると、`--single`で指定したグループには、パターン以外を追加しないことができる。逆に言えば、`--isolate`を指定しないと、パターンに一致しなかった残りのテストファイルは、等分配されることになる。


`options[:execute]`というのは、parallel に対して、コマンドラインからコードをしていたら、ということです。

サイトに解説がありましたので、引用します
```
> parallel_test -n 5 -e 'ruby -e "puts %[hello from process #{ENV[:TEST_ENV_NUMBER.to_s].inspect}]"'
> hello from process "2"
> hello from process ""
> hello from process "3"
> hello from process "5"
> hello from process "4"
```





```

[feature/parallel stash]~/projects/mugenup/workstation: bundle exec bin/rake parallel:spec[model] RAILS_ENV=test
4 processes for 55 specs, ~ 13 specs per process
...................................................................................................................................................................................................................................................................................................................................

Finished in 2 minutes 19.6 seconds
78 examples, 0 failures

Randomized with seed 2496

.................................................................................................

Finished in 3 minutes 19.1 seconds
124 examples, 0 failures

Randomized with seed 0

...................

Finished in 3 minutes 10.7 seconds
116 examples, 0 failures

Randomized with seed 43845

.....

Finished in 3 minutes 13.1 seconds
126 examples, 0 failures

Randomized with seed 58597


444 examples, 0 failures

Took 215.860779 seconds
[feature/parallel stash]~/projects/mugenup/workstation:
```

```
[feature/parallel stash]~/projects/mugenup/workstation: bundle exec bin/rake parallel:spec[model] RAILS_ENV=test
Using recorded test runtime
4 processes for 55 specs, ~ 13 specs per process
.......................................................................................................................................................................................................................................................................................................................................................................................................

Finished in 3 minutes 7.8 seconds
102 examples, 0 failures

Randomized with seed 0

......................

Finished in 2 minutes 58.3 seconds
108 examples, 0 failures

Randomized with seed 34146

...........................

Finished in 3 minutes 10.2 seconds
123 examples, 0 failures

Randomized with seed 50087

....

Finished in 3 minutes 11.8 seconds
111 examples, 0 failures

Randomized with seed 44916


444 examples, 0 failures

Took 214.314468 seconds
```

```
[feature/parallel stash]~/projects/mugenup/workstation: RAILS_ENV=test PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" bundle exec rake parallel:spec[model]
Using recorded test runtime
4 processes for 55 specs, ~ 13 specs per process
..........................................................................................................................................................................................................................................................................................................................................................................................................................

Finished in 2 minutes 59.2 seconds
102 examples, 0 failures

Randomized with seed 13490

........................

Finished in 3 minutes 9.4 seconds
109 examples, 0 failures

Randomized with seed 52806

....

Finished in 3 minutes 10.8 seconds
113 examples, 0 failures

Randomized with seed 17151

......

Finished in 3 minutes 14.3 seconds
120 examples, 0 failures

Randomized with seed 50440


444 examples, 0 failures

Took 216.075166 seconds
```

```
[feature/parallel stash]~/projects/mugenup/workstation: time RAILS_ENV=test PARALLEL_TESTS_EXECUTABLE="bundle exec rspec" bundle exec rake parallel:spec[model]
Using recorded test runtime
4 processes for 55 specs, ~ 13 specs per process
............................................................................................................................................................................................................................................................................................................................................................................................................................................

Finished in 3 minutes 6 seconds
105 examples, 0 failures

Randomized with seed 5216

......

Finished in 3 minutes 8.7 seconds
109 examples, 0 failures

Randomized with seed 44252

........

Finished in 3 minutes 13.2 seconds
115 examples, 0 failures

Randomized with seed 14684

..

Finished in 3 minutes 14.6 seconds
115 examples, 0 failures

Randomized with seed 28940


444 examples, 0 failures

Took 216.680576 seconds

real    3m47.661s
user    3m20.585s
sys     0m25.249s
```

```
[feature/parallel stash]~/projects/mugenup/workstation: time RAILS_ENV=test bundle exec bin/rake parallel:spec[model]
Using recorded test runtime
4 processes for 55 specs, ~ 13 specs per process
.................................................................................................................................................................................................................................................................................................................................................................................................................................

Finished in 3 minutes 16 seconds
109 examples, 0 failures

Randomized with seed 0

...............

Finished in 3 minutes 4.8 seconds
116 examples, 0 failures

Randomized with seed 20233

...........

Finished in 3 minutes 9.5 seconds
106 examples, 0 failures

Randomized with seed 51842

.

Finished in 3 minutes 1 second
113 examples, 0 failures

Randomized with seed 44906


444 examples, 0 failures

Took 212.844729 seconds

real    3m37.395s
user    0m1.378s
sys     0m0.285s
[feature/parallel stash]~/projects/mugenup/workstation:
```
