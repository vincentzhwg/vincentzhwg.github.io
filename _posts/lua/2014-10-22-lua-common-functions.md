---
title: lua常用函数备忘
date: 2014-10-22 14:53:00 +0800
tags:
- lua
- trim
---

### 字符串trim函数

    -- trim whitespace from both ends of string
    function trim(s)
        if s == nil then return nil else return s:find'^%s*$' and '' or s:match'^%s*(.*%S)' end
    end
    
    -- trim whitespace from left end of string
    function triml(s)
        if s == nil then return nil else return s:match'^%s*(.*)' end
    end
    
    -- trim whitespace from right end of string
    function trimr(s)
        if s == nil then return nil else return s:find'^%s*$' and '' or s:match'^(.*%S)' end
    end





