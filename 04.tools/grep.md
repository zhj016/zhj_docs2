
###rgrep search

#### find

  find . -name *.java | xargs grep static

#### rgrep
rgrep 指令的功能和grep 指令类似，可查找内容中包含指定的范本样式的文件。如果发现某文件的内容符合所指定的范本样式，预设rgrep指令会把含有范本样式的那一列显示出来。

  rgrep hello *

  grep -r "static" .

  -i ignore case
  -w word only
  -n line number
  -s no error
  -l list only file name
  -L list file names that not match
