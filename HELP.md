### Initialization and redevelopment

**Step1:**
```
cd <对应目录>
git checkout hexo
```
**Step2:**
```
npm install -g hexo-cli
npm install
```
**Step3:**
```angular2html
git submodule init
git submodule update
# 若子模块未正确加载
rm -rf themes/hexo-theme-matery
git clone https://github.com/blinkfox/hexo-theme-matery.git themes/hexo-theme-matery
```
**Step4:**
```angular2html
hexo clean
hexo g
hexo s
```

**Step4:**

```angular2html
# 代码更新后
git rm -r --cached -f themes/hexo-theme-matery
git submodule add https://github.com/blinkfox/hexo-theme-matery.git themes/hexo-theme-matery
# 推送静态文件
hexo clean
hexo deploy
```