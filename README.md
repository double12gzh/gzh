# 大海星的个人博客

## 本地测试

本博客通过ablog+sphinx+readthedocs实现，本地离线调试方式：

- 安装 ablog
- 安装 sphinx
- 运行命令

```bash
ablog clean
ablog build
ablog serve
```

## 自动构建

本地调试通过后，设置自动触发：

- 代码上传到github
- 注册[readthedocs](https://readthedocs.org/)帐号
- 在readthedocs导入项目

    ![1](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-readthedocs/readthedoc-1.png)

    ![2](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-readthedocs/readthedoc-2.png)

    ![3](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-readthedocs/readthedoc-3.png)

    ![4](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-readthedocs/readthedoc-4.png)

    ![5](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-20-readthedocs/readthedoc-5.png)

    > 说明: 如果最后一项没有设置的话，有可能会出现如下错误：
    >
    > ```bash
    > Extension error:
    > Handler <function process_postlist at 0x7f9c4b3d2510> for
    > event 'doctree-resolved' threw an exception
    > (exception: ('blogs/demo_in_root', None))
    > ```

## 参考

[Sphinx使用指南](https://www.cnblogs.com/double12gzh/p/13693395.html)