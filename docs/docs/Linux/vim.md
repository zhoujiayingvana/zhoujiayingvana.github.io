跳转第一行：gg

1. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">在 Normal 模式下，输入 </font>**<font style="background-color:rgb(247, 247, 248);">1G</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">，然后按下回车键。这将光标移动到第一行。</font>
2. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">在 Normal 模式下，输入 </font>**<font style="background-color:rgb(247, 247, 248);">:1</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">，然后按下回车键。这将光标移动到第一行。</font>

搜索

1. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">进入 Normal 模式：按下 </font>**<font style="background-color:rgb(247, 247, 248);">Esc</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);"> 键，确保你处于 Normal 模式。</font>
2. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">输入 </font>**<font style="background-color:rgb(247, 247, 248);">/</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">（斜杠）字符，紧接着输入你想要搜索的内容。例如，如果你想搜索单词 "example"，则输入 </font>**<font style="background-color:rgb(247, 247, 248);">/example</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">。</font>
3. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">按下回车键，Vim 将会跳转到第一个匹配到的结果。</font>
4. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">按下 </font>**<font style="background-color:rgb(247, 247, 248);">n</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);"> 键，可以继续向下搜索并跳转到下一个匹配结果。</font>
5. <font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">按下 </font>**<font style="background-color:rgb(247, 247, 248);">N</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);"> 键，可以向上搜索并跳转到上一个匹配结果。</font>

<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">此外，你也可以使用 </font>**<font style="background-color:rgb(247, 247, 248);">?</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">（问号）字符来进行反向搜索。例如，输入 </font>**<font style="background-color:rgb(247, 247, 248);">?example</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);"> 可以进行反向搜索。</font>

<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);">在搜索时，可以使用正则表达式来进行更复杂的匹配。如果需要更多的搜索选项和操作，请参考 Vim 的文档或者使用 </font>**<font style="background-color:rgb(247, 247, 248);">:help</font>**<font style="color:rgb(55, 65, 81);background-color:rgb(247, 247, 248);"> 命令在 Vim 中查看帮助文档。</font>

