---
title: lua常用函数备忘
date: 2014-10-22 14:53:00 +0800
tags:
- lua
- trim
---

* toc
{:toc}


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






### 字符串切割函数

    function string_split(str, split_char)
        local sub_str_tab = {}
        local i = 1
        local j
        while true do
            j = string.find(str, split_char, i)
            if j == nil then
                table.insert(sub_str_tab, string.sub(str, i))
                break
            end
            table.insert(sub_str_tab, string.sub(str, i, j - 1))
            i = j + 1
        end
        return sub_str_tab
    end

