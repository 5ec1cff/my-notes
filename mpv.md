# mpv 调教日记

[mpv](https://mpv.io) 是一款自由的媒体播放器。

自从某一天打开 potplayer 后发现桌面右下角不知哪来的韩文广告后，我就对它敬而远之了，实际上 potplayer 的恶名也早有耳闻，于是转向了自由开源的 mpv 。然而虽然开源自由，mpv 却并非那么容易上手，需要经过一定的调教，才能改造成自己喜欢的模样。

## 配置文件

在非 Windows 上，默认配置目录位于 `~/mpv` ，而在 Windows 上则是 `%AppData%` 。一开始以为是 `%UserProfile%` ，这样有点反直觉。

这个目录下，`mpv.conf` 包含了播放器的默认配置，实际上就是默认命令行选项，去掉了前面的 `--` 。

## 窗口默认大小

```
autofit=1024
```

## 控制台

按反引号（<code>&#96;</code>）即可打开控制台

## 字幕加载

简单的：

```
# 精确匹配名字
# sub-auto=exact
# 加载同目录下所有字幕
sub-auto=all
```

但是这样不能加载次字幕（secondary subtitles）。其实 potplayer 也不能，每次都要手动添加。

某些字幕组的外挂字幕会分成两部分（次字幕的名字包含 `Annotations`）。

mpv 的*简陋* UI 可以切换字幕，然而不能选择次字幕。

使用 mpv 的脚本功能，勉强实现了这一需求。

脚本放在配置目录的 `scripts` 子目录下。

```js
mp.add_hook("on_preloaded", 0, function() {
    var p = mp.get_property("path");
    mp.msg.info(p);
    var file_name = mp.utils.split_path(p)[1];
    var name = file_name.match(/(.*)\.(.*?)/)[1];

    var n = mp.get_property_number("track-list/count");
    var sub = null, sub2 = null;
    for (var i = 0; i < n; i++) {
        var track_id = mp.get_property_number("track-list/" + i + "/id");
        var track_type = mp.get_property("track-list/" + i + "/type");
        var track_filename = mp.get_property("track-list/" + i + "/external-filename");
        if (track_type == 'sub') {
            // secondary sub contains 'Annotation'
            if (track_filename.match(/Annotation/i)) {
                if (sub2 == null) {
                    sub2 = track_id;
                }
            } else {
                // we prefer the one whose name is similar to video file's name
                if (sub == null || track_filename.indexOf(name) != -1) {
                    sub = track_id;
                }
            }
        }
    }
    mp.msg.info("sub=" + sub);
    mp.msg.info("sub2=" + sub2);
    if (sub !== null) {
        mp.set_property("options/sid", sub.toString());
    }
    if (sub2 !== null) {
        mp.set_property("options/secondary-sid", sub2.toString());
    }
})
```

mpv 的事件处理不能感知新文件的添加，因此无法实现拖放字幕后处理的情况，因此需要选项 `sub-auto=all` 在文件打开的时候加载所有字幕，在 `on_preloaded` hook 点处理，使用 `set_property` 选择字幕和次字幕。
