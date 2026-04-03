# mpv.lazycmd

通用 `mpv` 后台播放器插件。进入 `/mpv` 后展示当前后台播放列表，也可以被其他音乐插件通过 `require('mpv')` 复用。

## 功能

- `/mpv` 直接显示当前 `mpv` IPC 播放队列
- 后台 `mpv` 切歌后，如果当前就在 `/mpv`，列表会自动把悬停项移动到正在播放的歌曲
- 如果 `mpv` 未启动，首次播放时自动拉起后台 `mpv`
- 支持播放控制：跳转到当前项、暂停/继续、上一首、下一首、恢复播放、调节音量
- 支持从 `/mpv` 队列删除指定歌曲
- 其他插件可以调用 `require('mpv').play_tracks()` / `append_tracks()` 往队列里塞歌
- 其他插件在加歌时可以为每个条目传入自己的 `keymap` 和 `preview`
- 公开异步 API 统一返回 promise
- 调用方必须先执行 `require('mpv').setup(...)`，插件不再自动从 `plugins` 配置里推断初始化

## 配置

```lua
{
  dir = 'plugins/mpv.lazycmd',
  config = function()
    require('mpv').setup {
      socket = '/tmp/lazycmd-mpv.sock',
      mpv_args = {
        '--idle=yes',
        '--no-video',
        '--force-window=no',
        '--audio-display=no',
        '--really-quiet',
      },
      keymap = {
        jump = '<enter>',
        pause = '<space>',
        next = 'n',
        prev = 'p',
        delete = 'dd',
        volume_up = '+',
        volume_down = '-',
      },
    }
  end,
},
```

别的模块直接 `require('mpv')` 后，必须先调用 `setup()`，否则公开 API 会报错。

## 提供的 Lua API

### `mpv.play_tracks(tracks)`

替换当前队列并开始播放。
返回一个 promise。

### `mpv.append_tracks(tracks)`

追加到当前队列尾部并开始播放。
返回一个 promise。

### `mpv.update_track_fields(id, fields)`

按 `id` 更新队列内缓存元数据，适合自定义 `display/preview` 依赖的轻量字段刷新。

### `mpv.get_player_state()`

返回当前播放器状态：

```lua
{
  running = true,
  pause = false,
  playlist = { ... },
}
```

返回一个 promise。

### `mpv.player_remove(index)`

按 `mpv` 播放列表下标删除指定条目。
返回一个 promise。

## Track 结构

`play_tracks()` 和 `append_tracks()` 接收的每个 track 至少需要 `url`：

```lua
{
  id = song.id,
  url = stream_url,

  -- 可选：自定义队列展示
  display = function(item, player, meta) ... end,

  -- 可选：自定义队列预览
  preview = function(entry, cb) ... end,

  -- 可选：自定义 entry 级快捷键
  keymap = {
    ['s'] = { callback = function() ... end, desc = 'toggle star' },
  },
}
```

`mpv` 会自动把播放器控制键和你传入的 `keymap` 合并。

默认列表只显示 `mpv` 自己能通过 socket 返回的条目属性，例如 `title`、`filename`、`current/playing`。默认 preview 为空；如果需要更丰富的展示，由调用方通过 `display` 或 `preview` 自定义。

内置控制键位默认是：

- `jump`: 跳到当前队列项
- `pause`: 暂停/继续
- `next`: 下一首
- `prev`: 上一首
- `delete`: 从队列删除当前项
- `volume_up`: 增大音量
- `volume_down`: 减小音量
