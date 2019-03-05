slidenumbers: true

# [fit] Automate all the things

---

# [fit] あらゆる作業を自動化しよう

---

# Self Intro

- Joshua Kaplan
- minne @ GMO Pepabo
- Danger-Swift contributer
- Likes 🤖

^ 日本最大のハンドメイドマーケットプレイスである minne の iOS エンジニア
本日発表する Danger の contributer
機械に仕事をしてもらうのが好き

---

# [fit] CI is the ultimate automation tool

^ CI が最強の自動化ツールである

---

# [fit] But there are a range of other tools

---

- GitHub Actions and Apps
- Bots

---

# [fit] Why automate?

---

- Fewer mistakes
- Faster
- More efficient
- More exciting

* Goals
  _ Improve efficiency
  _ General tenet of economic theory
  _ “Emotion” and coaching can only get you so far
  _ Prevent mistakes
  _ Increase development speed
  _ Improve stability and reliability

^ App Store Connect にログインするのが好きな人いる？

---

# [fit] What areas to automate?

---

- Take time for _multiple people_
- High risk
- Prone to being forgotten

^(it’s not just yourself doing that task 15 minutes a month
^(eg: major bugs w/ calendar or answering a question wrong w/ App Submission)
^ (closing the milestone and making release notes)

---

# Key CI tools

- Danger and other automation platforms
- Linters
- ????

---

# [fit] Normal CI flow (level 1)

---

# Run tests for each PR

^ Github につなげて、単体テストをすべての PR に対して実行する
これを code review と組み合わせて、品質を担保する

---

![100%](images/test_results.png)

---

# [fit] Intermediate CI flow (level 2)

^ それだけではない

---

# [fit] Linters and other static analysis

^ linter などの静的解析ツールにより、さらなる品質とレビューの自動化を実現できる

---

# SwiftLint + Danger

^ これだいたい知られているやつ

```rb
swiftlint.lint_files inline_mode: true
```

---

![100%](images/danger_warning.png)

---

# Original static analysis

^ storyboard の簡易な linter も行える
本当はツールあるが、とりあえずこれだけでよい

---

```rb
# storyboardにずれているビューや制約の不備がないか
parsed_diff.each do |changed_file|
  next unless changed_file.file.match(/.*\.(storyboard)$/)

  changed_file.changed_lines.each do |changed_line|
    content = changed_line.content
    has_misplaced_view = content.include? 'misplaced="YES"'
    has_ambiguous_view = content.include? 'ambiguous="YES"'

    file = changed_file.file
    line_number = changed_line.number

    if has_misplaced_view
      fail('ずれているビューがあるので、Update Framesを実行してください', file: file, line: line_number)
    end
    if has_ambiguous_view
      fail('制約が不十分な箇所があるので、直してください', file: file, line: line_number)
    end
  end
end
```

---

# Spell checking

^ 私が職場で唯一の英語ネーティブなので、スペルに厳しいやつとして見られた
スペルの間違いは私がいつも気づくからだ
エンジニアは機械に怒られることにとっても慣れているので、ボットにやらせることで、自分の厳しい印象を払拭して、自分の仕事を楽にした
cspell という JS のツールを使っている. Xcode には spell check のメニュー項目はあるが、機能しない

---

![100%](images/spell_check.png)

---

```rb
# タイポを検知する
added_and_modified_files = git.added_files + git.modified_files
added_and_modified_files.each do |file_path|
  next unless file_path =~ /\.swift$/
  stdout, status = Open3.capture2("npx", "cspell", file_path)

  next if status.success?
  stdout.split("\n").each do |line|
    next unless matches = /\w+\.swift:(\d+).*-\sUnknown\sword\s\((\w+)\)/.match(line)
    line_number = matches[1].to_i
    word = matches[2]

    warning = "タイポかも？ #{word}"
    warn(warning, file: file_path, line: line_number)
  end
end
```

---

# [fit] Automate PR tasks

---

# Assign reviewers

---

```rb
# レビュワー指定（コメント指定のためにこれを一番上に書く必要あり）
reviewers = ["user1", "user2", "user3"].reject { |reviewer| reviewer == github.pr_author }
repo_name = github.pr_json["head"]["repo"]["full_name"]
pr_number = github.pr_json["number"]

number_of_comments = github.api.issue_comments(repo_name, pr_number).size
if number_of_comments.zero?

  reviewers = reviewers.sample(2)

  github.api.request_pull_request_review(
    repo_name,
    pr_number,
    {},
    "reviewers": reviewers
  )
end
```

---

# Post CI results

^ CI のログを見に行く時間を減らすと楽
また、某 B 社のちょっと遅いログのロード時間を省ける

---

![100%](images/xcode_summary.png)

---

```rb
# Xcode Summary
build_report_file = 'build_results.json'
xcode_summary.ignored_files = 'Pods/**'
xcode_summary.ignores_warnings = true
xcode_summary.inline_mode = true
xcode_summary.report build_report_file
```

---

# Post code coverage

^ 絵文字でテストを書くモチベーションをあげる
また、最低のカバレッジを設定できるので、品質管理の面でも効果的

---

![100%](images/xcov_danger.png)

---

```rb
xcov.report(
  scheme: 'minne',
  workspace: 'minne.xcworkspace',
  exclude_targets: 'TodayExtension.appex, NotificationServiceExtension.appex, MinneKit.framework',
)
```

---

# [fit] Advanced CI flow (level 3)

---

# Custom linter rules

^ すべての会社には、特定のルールやベストプラクティスがあるので、custom ルールでそのルールの遵守を担保できる

---

```yml
prefer_gregorian_calendar:
  name: "Gregorian Calendar"
  regex: "Calendar\\.current"
  message: "Please use `Calendar(identifier: .gregorian)` to avoid Japanese calendar-related bugs"
  severity: error
https_only:
  name: "HTTPS Only"
  match_kinds:
    - string # コメントなどを無視して、文字列のみをみる
  regex: "http:"
  message: "Please use HTTPS due to security policy"
  severity: error
```

---

# Automatic dependency updates

---

# Deployment

---

```rb
lane :release do
  capture_screenshots                  # generate new screenshots for the App Store
  sync_code_signing(type: "appstore")  # see code signing guide for more information
  build_app(scheme: "MyApp")
  upload_to_app_store                  # upload your app to App Store Connect
  slack(message: "Successfully uploaded a new App Store build")
end
```

---

# [fit] "Liquid Software"

---

# AKA Continuous Deployment

---

# [fit] Speed optimization

---

# Caching

- https://github.com/blender/Rome

^ CI の時間を半分にした

---

# Parallelization

---

# [fit] Bots

---

# ProBot does

- [Probot](https://probot.github.io/docs/development/)
- Automatically marking old branches as stale
- Reminders
- Delete merged branch
- Marking issues to be closed when PR is merged

---

# [fit] Conclusion

---

# [fit] Automate all the things
