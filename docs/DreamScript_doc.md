# DreamScript

> 此文档为设计预案，并不代表最终标准，欢迎在 [Issue](https://github.com/EtherProject/DreamFramework/issues) 中提交建议

## 关于 DreamScript

+ DreamScript 是为 DreamFramework 设计的领域专用语言
+ DreamScript 只负责制定演出顺序及组织结构，具体逻辑由 Lua 脚本实现
+ DreamScript 在开发环境下解析运行，发布时将编译为二进制形式的字节码
+ DreamScript 的脚本文件默认扩展名为 `.ds`（大小写不敏感），这对于 解析器 / 编译器 而言是很重要的

## 语法及示例

+ 对话脚本：
    - 按行解析
    - 说话者与内容使用 `*` 分隔，`*` 前后的空格将会被忽略
    - 如果需要在文本中使用 `*`，请使用 `**` 替换一个 `*`
    - 说话者或者内容均可以省略
    
    示例如下：

```
老八 * 今天小汉堡买一赠一
* 听到这句话，杯中82年的拉菲突然就不香了

李华 * 我**尼玛，又让我给国外网友写信
```

> 考虑到 DreamScript 除去基于可视化工具自动生成之外，部分编剧可能倾向于直接手动编写演出脚本，那么在 `说话者` 与 `内容` 之间的分隔符是需要慎重选取的，因为考虑到分隔符的特殊含义，所以不能选取在文本中常见的字符；以及，考虑到不同语言文本的书写习惯，无论是使用 `[]` 还是 `【】`，相信都会导致一批开发者在切换 半/全角 是怨声载道
>
> 而转义符 `\` 并不符合常规思维（并且同样会出现上述的 半/全角 切换问题），而且在非程序领域，使用转义符的应用场景并不多，所以考虑使用重复的符号代表符号本身的含义

+ 注释：
    - 不区分单行注释和多行注释
    - 注释内容包裹在 `%{ }%` 中
    - 不支持注释嵌套

    示例如下：
    
```
%{
    言语粗俗，
    可能会对未成年人产生不良影响，
    后续联系策划修改人物性格设定
}%
李华 * 我**尼玛，又让我给国外网友写信
```

> 由于注释并不如剧本文案那般占据大部分内容，所以写法稍微负责一下 ——
>
> 另外，应该不会有人会在注释内容里面写 `%{ }%` 这种奇奇怪怪的东西吧……

+ 指令：
    - 指令是游戏的逻辑所在，画面演出、音频播放、存档处理 等功能均由指令提供
    - 指令以 `@` 或 `$` 开头，紧接着的是 Lua 脚本中对应模块的函数名及参数列表，参数传递遵循 Lua 语法
    - 可以在 Lua 脚本中使用 `RegisterToGlobal` 函数将自定义的脚本注册到全局命令中
    - `@` 修饰的指令遵循向下绑定的原则，即当前行的指令是在下一行开始前被执行
    - `$` 修饰的指令为 `立即指令`，即当前行指令在前一行结束后就被执行
    - 在脚本开始处的指令无论是否为立即指令都会被立刻执行

    示例如下：

```lua
-- Script/Command/MyCommand.lua

-- 将一个匿名函数注册为全局指令
-- 全局指令不受模块命名空间限制，可以直接索引
RegisterToGlobal(
    "AChallenge",
    function(str)
        print("一个挑战 ……")
        print(str)
        print("你输了！")
    end
)

return {
    -- 让游戏发出 “哼哼嗯 —— 啊啊啊啊啊啊” 的野兽叫声
    MakeGameAAAAAa = function()
        -- 调用 DreamFramework 内置 API
        local _music = LoadMusic("TheBestMusic.mp3")
        SetGameVolume(100)
        PlayMusic(_music)
    end
}
```

```
%{
    此时正在播放的是 TheBestMusic.mp3 ……
}%
@MyCommand.MakeGameAAAAAa()
李田所 * 律师函正在路上！

%{
    调用 DreamFramework 的全局内置函数将 lihua.png 以 200 x 750 的分辨率显示在屏幕中间
    并调用注册为全局函数的自定义函数，控制台会输出如下内容：
    "一个挑战 ……"
    "别看控制台"
    "你输了！"
}%
@ShowMidChara("lihua.png", 200, 750)
@AChallenge("别看控制台")
李华 * 想不到除了英语作文还有可以折磨你的东西！
```

+ 标签：
    - 标签是为剧本跳转锚定设计的，配合内置全局指令 `@JumpToLabel` 使用
    - 标签以 `#` 开头，并且占据单独的一行
    - 标签是全局可见性，即可以实现在不同脚本文件间进行跳转
    - 请务必确保标签名的唯一性，重名的标签可能会发生覆盖，并且发生不可预知的执行顺序

    示例如下：

```
%{ 一直做梦…… }%

# dream_start 1
@JumpToLabel("dream_start 2")
* 梦开始的地方

# dream_start 2
@JumpToLabel("dream_start 1")
```

+ 代码块：
    - 代码块即直接嵌入在 DreamScript 中的 Lua 脚本，使用 `[@[ ]@]` 或 `[$[ ]$]` 包裹，不区分单行与多行
    - 与指令类似，`[@[ ]@]` 代码块的执行时间会向下绑定，而 `[$[ ]$]` 则向上绑定
    - DreamScript 的设计思想是逻辑与演出分离，所以并不建议在脚本中直接编写代码块
    - 代码块在开发阶段可以用来输出变量和环境信息，辅助调试，所以保留此语法
    - 注意，由于跳转的存在，同一处的代码块可能会被多次执行，注意防止代码的副作用

    示例如下：

```
%{ 可以起到和前面相同的作用…… }%
[@[
    MyCommand.MakeGameAAAAAa()
]@]
李田所 * 律师函正在路上！
```

> 由于 Lua 支持使用 `[[]]` 或 `[=[]=]` 等包裹的多行文本，所以就只好使用这种符号作为代码块的标志了
>
> 相信也不会有人会在代码里面写 `[@[ ]@]` 或 `[$[ ]$]` 这种东西吧，不会吧不会吧？！

## 其他

+ DreamScript 遵循的原则是按行解析，除非遇到 `代码块` 或 `注释块` 等支持多行的元素，在大多数情况下，行内 **标识符前后** 的空字符均是被 解析器 / 编译器 忽略的，不会引发异常或解析错误，如下：

```
%{
    如下三种写法均可以被 DreamFramework 所接受
    但是在下定决心这样做之前，建议先测试一下制作组内程序负责人的心理承受能力
}%
@MyCommand.MakeGameAAAAAa()
    @MyCommand.MakeGameAAAAAa()
@   MyCommand.MakeGameAAAAAa()

%{ 如下的写法，标签名最终都会被解析为 "dream_start 1" }%
#dream_start 1
    #dream_start 1
#   dream_start 1
```