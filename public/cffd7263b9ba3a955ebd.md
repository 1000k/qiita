---
title: grunt watch が遅い時の改善方法
tags:
  - JavaScript
  - grunt
private: false
updated_at: '2015-06-21T16:41:36+09:00'
id: cffd7263b9ba3a955ebd
organization_url_name: null
slide: false
ignorePublish: false
---
Grunt を使うからには [grunt-contrib-watch](https://github.com/gruntjs/grunt-contrib-watch#optionsnospawn) でファイルの変更を監視して自動ビルドしたいのですが、どうも遅い。

```bash
$ grunt watch

Running "watch" task
Waiting...OK
>> File "src/sass/screen.scss" changed.

Running "compass:dist" (compass) task
overwrite app/css/screen.css (0.333s)
Compilation took 0.342s

Running "cssmin:add_banner" (cssmin) task
File app/css/screen.min.css created.

Running "cssmin:minify" (cssmin) task
File app/css/app.min.css created.
File app/css/screen.min.css created.

Done, without errors.
Completed in 12.602s at Sat Oct 19 2013 16:26:32 GMT+0000 (UTC) - Waiting...
```

タスクの実行自体は1秒もかかっていないのに、全体では12秒ほどかかっていました。

Gruntfile の設定を見直すことで改善できたので、そのやり方をメモしておきます。


<!--more-->


`grunt watch` のボトルネックは、ファイルの変更を検知後にタスクプロセスを子プロセスとしてスポーンさせる部分にあるようです。
したがって、子プロセスとしてタスクを起動するオプションを無効にすることで回避できます。

以下のように、Gruntfile.js 内で `spawn: false` を指定してやれば OK です。

```js
module.exports = function(grunt) {
  grunt.initConfig({
    // ...
    watch: {
      options: {
        spawn: false
      },
      css: {
        files: ['src/sass/*.scss'],
        tasks: ['compass', 'cssmin']
      }
    }
  });
  // ...
```

これで大幅にスピードが改善されました。

```bash
$ grunt watch

Running "watch" task
Waiting...OK
>> File "src/sass/screen.scss" changed.


Running "compass:dist" (compass) task
identical app/css/screen.css (0.319s)
Compilation took 0.328s

Running "cssmin:add_banner" (cssmin) task
File app/css/screen.min.css created.

Running "cssmin:minify" (cssmin) task
File app/css/app.min.css created.
File app/css/screen.min.css created.

Running "watch" task
Completed in 0.712s at Sat Oct 19 2013 16:27:58 GMT+0000 (UTC) - Waiting...
OK
>> File "src/sass/screen.scss" changed.
```

12秒が0.7秒に。イェーイ！


参考
----
- [Why Watch is so slow compared to Regarde · Issue #69 · gruntjs/grunt-contrib-watch](https://github.com/gruntjs/grunt-contrib-watch/issues/69)
- [gruntjs/grunt-contrib-watch#optionsnospawn](https://github.com/gruntjs/grunt-contrib-watch#optionsnospawn) 
