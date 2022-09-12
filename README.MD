### install hexo 
* npm install -g hexo
* npm install hexo --save
* hexo -v . check install success or not

### hexo command 
* hexo init 
* hexo s , run server in local

### how to upload to github
   * add config in file `_config.yml` 
```
deploy:
  type: git
  repository: git@github.com:Yuck1125/Yuck1125.github.io.git
  branch: master
```

* npm install hexo-deployer-git --save
* hexo g
* hexo d