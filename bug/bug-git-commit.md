| title          | tags   | created               | modified              |
|----------------|--------|-----------------------|-----------------------|
| bug-git-commit        | [blog] | 2025/3/18 13:24| 2025/3/18 13:24 |

[2024-3-18]
来源：https://hk.v2ex.com/t/1119177#reply3
问题： 代码提交到了本地仓库，使用 github desktop 的 add local repository 添加本地仓库，点击 pulish repository ，设置仓库名以及可见性都没问题，但推送代码发现无法点击，发现分支可以切换（但我本地只有一个分支），切换后推送上去了，此时再打开 vscode ，就发现本地仓库最近一周的提交记录都没了，github 上也没有，怎么找回呢？
解决方法：git reflog --date=iso