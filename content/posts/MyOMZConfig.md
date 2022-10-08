---
title: "我的omz主题配置"
date: 2022-10-08T18:08:15+08:00
draft: false
---

习惯了zsh就很难再习惯别的shell了2333.

omz主题配置目录为`～/.oh-my-zsh/themes/`, 我的主题是一个简单修改过的agnoster.

修改路径长度：
``` bash
# Dir: current working directory
prompt_dir() {
    prompt_segment blue $CURRENT_FG '%25<...<%~%<<'
    #prompt_segment blue $CURRENT_FG '%~'
}
```

添加Conda配置:
``` bash
# Conda
prompt_conda() {
  if [[ -n "$VIRTUAL_ENV" ]]; then
    prompt_segment yellow $CURRENT_FG $VIRTUAL_ENV 
  elif [[ -n "$CONDA_DEFAULT_ENV" ]]; then
    prompt_segment green $CURRENT_FG $CONDA_DEFAULT_ENV
  fi
}
```