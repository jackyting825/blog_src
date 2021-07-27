---
title: js检测浏览器页面是否缩放
date: 2021-07-27 10:52:39
tags: 
  - js
---

>使用js前端检测浏览器的页面是否手工缩放

```javascript

  /**
   *
   * detectZoom 函数的返回值如果是 100 就是默认缩放级别，大于 100 则是放大了，小于 100 则是缩小了
   **/
  function detectZoom (){ 
    var ratio = 0,
      screen = window.screen,
      ua = navigator.userAgent.toLowerCase();
  
    if (window.devicePixelRatio !== undefined) {
        ratio = window.devicePixelRatio;
    }
    else if (~ua.indexOf('msie')) {  
      if (screen.deviceXDPI && screen.logicalXDPI) {
        ratio = screen.deviceXDPI / screen.logicalXDPI;
      }
    }
    else if (window.outerWidth !== undefined && window.innerWidth !== undefined) {
      ratio = window.outerWidth / window.innerWidth;
    }
    
    if (ratio){
      ratio = Math.round(ratio * 100);
    }
    
    return ratio;
  }

```