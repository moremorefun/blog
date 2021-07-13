---
author: admin
comments: true
date: 2013-11-28 12:27:24+00:00
layout: post
slug: cocos2d-x3-0%e5%88%a9%e7%94%a8stdfunction%e4%bd%bf%e7%94%a8actiontween
title: cocos2d-x3.0利用std::function使用ActionTween
wordpress_id: 288
categories:
- cocos2d-x
tags:
- '3.0'
- ActionTween
---

cdx提供了任意属性的Action回调,
但是runAction的对象需要实现ActionTweenDelegate接口.
鉴于3.0使用了`c++11`,而`c++11`提供了`std::function`,
我们为什么不能利用回调来代替接口实现任意属性的缓动动作呢?
无非是传一个`std::function`保存在Action里而已,基本代码和ActionTween没有太大区别.

```c++
#ifndef __ActionTweenWithFunc_H__
#define __ActionTweenWithFunc_H__

#include "CCActionInterval.h"

class ActionTweenWithFunc : public cocos2d::ActionInterval
{
public:
    static ActionTweenWithFunc* create(float duration, std::function<void(float cur_num)> func, float from, float to);

    bool initWithDuration(float duration, std::function<void(float cur_num)> func, float from, float to);

    // Overrides
    void startWithTarget(cocos2d::Node *target) override;
    void update(float dt) override;
    ActionTweenWithFunc* reverse() const override;
    ActionTweenWithFunc* clone() const override;
protected:
    std::function<void(float cur_num)>  _up_func;
    float            _from, _to;
    float            _delta;
};

#endif /* __ActionTweenWithFunc_H__ */


#include "ActionTweenWithFunc.h"

ActionTweenWithFunc* ActionTweenWithFunc::create(float aDuration, std::function<void(float cur_num)> func, float from, float to)
{
    ActionTweenWithFunc* pRet = new ActionTweenWithFunc();
    if (pRet && pRet->initWithDuration(aDuration, func, from, to))
    {
        pRet->autorelease();
    }
    else
    {
        CC_SAFE_DELETE(pRet);
    }
    return pRet;
}

bool ActionTweenWithFunc::initWithDuration(float aDuration, std::function<void(float cur_num)> func, float from, float to)
{
    if (ActionInterval::initWithDuration(aDuration))
    {
        _up_func = func;
        _to       = to;
        _from     = from;
        return true;
    }
    return false;
}

void ActionTweenWithFunc::startWithTarget(cocos2d::Node *target)
{
    ActionInterval::startWithTarget(target);
    _delta = _to - _from;
}

void ActionTweenWithFunc::update(float dt)
{
    float cur_num = _to - _delta * (1 - dt);
    _up_func(cur_num);
}

ActionTweenWithFunc* ActionTweenWithFunc::reverse() const
{
    return ActionTweenWithFunc::create(_duration, _up_func, _to, _from);
}

ActionTweenWithFunc* ActionTweenWithFunc::clone() const
{
    return ActionTweenWithFunc::create(_duration, _up_func, _from, _to);
}

```
