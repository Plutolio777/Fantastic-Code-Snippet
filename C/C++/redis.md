# 摘自redis-1.6.2

该段代码是redis中用于读取用户输入命令的代码

~~~C
static char *prompt(char *line, int size) {
    char *retval;
    do {
        printf(">> ");
        // 读取输入
        retval = fgets(line, size, stdin);
        // 如果输入仅是回车的话则重复循环
    } while (retval && *line == '\n');
    // 转换成字符串
    line[strlen(line) - 1] = '\0';
    return retval;
}
~~~
