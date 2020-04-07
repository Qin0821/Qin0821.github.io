博客源代码分支

更换主题
· clone主题到该分支
ex： git clone -b master https://github.com/wujun234/hexo-theme-tree themes/tree
· 更改_config.yml
ex：theme：tree
· config
· upgrade 
ex：cd themes/tree
git pull

发布到Github
· hexo g 生成静态文件
· hexo d 发布到github，会根据_config.yml中deploy相关配置上传到指定分支（master）
· 提交本地代码到hexo分支以供备份