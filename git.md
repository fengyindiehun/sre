# 关于git工作流的胡言乱语

## 环境准备

#### github准备工作

1. github[开通二次验证](https://github.com/settings/two_factor_authentication/configure)
2. github配置账户[SSH key](https://github.com/settings/keys)

#### 本地环境初始化

以本项目为例：

1. 推荐配置一下本地的 ~/.gitconfig
```
[color]
    ui = auto
[alias]
    lg1 = log --graph --all --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(bold white)— %an%C(reset)%C(bold yellow)%d%C(reset)' --abbrev-commit --date=relative
    lg2 = log --graph --all --format=format:'%C(bold blue)%h%C(reset) - %C(bold cyan)%aD%C(reset) %C(bold green)(%ar)%C(reset)%C(bold yellow)%d%C(reset)%n''          %C(white)%s%C(reset) %C(bold white)— %an%C(reset)' --abbrev-commit
[core]
    editor = vim
    safecrlf = true
    excludesfile = ~/.gitignore
[push]
    default = current
[rerere]
    enabled = 1
    autoupdate = 1
[user]
    name = 改为你自己的昵称
    email = 改为你github上验证过的邮箱地址
[merge]
    tool = vimdiff
[url "git@github.com:"]
    insteadOf = https://github.com/
[url "git@github.com:"]
    insteadOf = http://github.com/
[url "git@github.com:"]
    insteadOf = git://github.com/
```

2. 先在github上[本项目页面](https://github.com/eleme/sre)fork一记。
3. 然后本地clone项目并且把自己fork的repo作为另外一个remote添加到本地配置中。

```
git clone git@github.com:eleme/sre.git
cd sre
git remote add self git@github.com:yourself/sre.git
```

## 新功能开发

首先明确你需要基于哪个分支进行开发，并更新所需分支到最新。如果是需要基于eleme上面的master分支进行功能开发，则直接更新本地master到最新

```
git checkout master
git pull
```

更新完master后，切到一个新的分支开始功能开发

```
git checkout new_feature
```

在这个新的功能分支你会做一些功能的开发，每做一个小的功能点就commit一下，保证commit的原子性能方便其他人来review你的代码。

```
git add bar.md
git commit -m "添加bar文档"
git add foo.md
git commit -m "添加foo文档"
```

在你做完所需的功能开发后，可以将代码推到自己的仓库去

```
git push self
```

最后你可以在你github上自己的repo里看到你最新的分支，然后提交一个Pull Request到你切出来的分支即可。
