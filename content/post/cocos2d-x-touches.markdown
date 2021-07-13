---
author: admin
comments: true
date: 2013-08-29 15:14:51+00:00
layout: post
slug: cocos2d-x%e7%9a%84touches%e7%9a%84%e5%a4%84%e7%90%86
title: cocos2d-x的touches的处理
wordpress_id: 271
categories:
- cocos2d-x
tags:
- touches
---

经过不同平台的适配,cdx的touches处理是从
`cocos2dx/platform/CCEGLViewProtocol.cpp::handleTouchesBegin(int num, int ids[], float xs[], float ys[])`开始的,
传进来的参数是一个touch的点数,touch的id, x, y的数组

```c++
void CCEGLViewProtocol::handleTouchesBegin(int num, int ids[], float xs[], float ys[])
{
    CCSet set;
    for (int i = 0; i < num; ++i)
    {
        //遍历所有传进来的touch点信息
        int id = ids[i];
        float x = xs[i];
        float y = ys[i];
        //从touch存储字典中查找当前touch点的信息
        CCInteger* pIndex = (CCInteger*)s_TouchesIntergerDict.objectForKey(id);
        int nUnusedIndex = 0;

        // it is a new touch
        if (pIndex == NULL)
        {
            //touch字典中没有当前的touch点数据
            //获取一个touch的标示
            nUnusedIndex = getUnUsedIndex();

            // The touches is more than MAX_TOUCHES ?
            if (nUnusedIndex == -1) {
                CCLOG("The touches is more than MAX_TOUCHES, nUnusedIndex = %d", nUnusedIndex);
                continue;
            }

            CCTouch* pTouch = s_pTouches[nUnusedIndex] = new CCTouch();
            //初始化本地touch对象的数据
           pTouch->setTouchInfo(nUnusedIndex, (x - m_obViewPortRect.origin.x) / m_fScaleX,
                                     (y - m_obViewPortRect.origin.y) / m_fScaleY);

            //CCLOG("x = %f y = %f", pTouch->getLocationInView().x, pTouch->getLocationInView().y);

            CCInteger* pInterObj = new CCInteger(nUnusedIndex);
            //将touch储存在字典中
            s_TouchesIntergerDict.setObject(pInterObj, id);
            set.addObject(pTouch);
            pInterObj->release();
        }
    }

    if (set.count() == 0)
    {
        CCLOG("touchesBegan: count = 0");
        return;
    }
    //交给touch时间处理器处理这些本地化以后的touch对象
    m_pDelegate->touchesBegan(&set, NULL);
}
```

继续跟踪`m_pDelegate->touchesBegan(&set, NULL)`的处理在`cocos2dx/touch_dispatcher/CCTouchDispatcher.cpp`中

```c++
void CCTouchDispatcher::touchesBegan(CCSet *touches, CCEvent *pEvent)
{
 if (m_bDispatchEvents)
 {
     //交给了本来的touches函数处理,参数pEvent实际是NULL
     //第三个参数是事件标示,begin, move, end
     this->touches(touches, pEvent, CCTOUCHBEGAN);
 }
}
```

跟下去

```c++
void CCTouchDispatcher::touches(CCSet *pTouches, CCEvent *pEvent, unsigned int uIndex)
{
    CCAssert(uIndex >= 0 && uIndex < 4, "");

    CCSet *pMutableTouches;
    //一个标示,当true的时候不允许直接添加新的handler对象
    m_bLocked = true;

    // optimization to prevent a mutable copy when it is not necessary
     unsigned int uTargetedHandlersCount = m_pTargetedHandlers->count();
     unsigned int uStandardHandlersCount = m_pStandardHandlers->count();
    bool bNeedsMutableSet = (uTargetedHandlersCount && uStandardHandlersCount);

    pMutableTouches = (bNeedsMutableSet ? pTouches->mutableCopy() : pTouches);

    struct ccTouchHandlerHelperData sHelper = m_sHandlerHelperData[uIndex];
    //
    // process the target handlers 1st
    //
    if (uTargetedHandlersCount > 0)
    {
        CCTouch *pTouch;
        CCSetIterator setIter;
        for (setIter = pTouches->begin(); setIter != pTouches->end(); ++setIter)
        {
            //遍历所有传进来的本地化touch对象
            pTouch = (CCTouch *)(*setIter);

            CCTargetedTouchHandler *pHandler = NULL;
            CCObject* pObj = NULL;
            CCARRAY_FOREACH(m_pTargetedHandlers, pObj)
            {
                //遍历所有已经注册的toucheHandler
                pHandler = (CCTargetedTouchHandler *)(pObj);

                if (! pHandler)
                {
                   break;
                }

                bool bClaimed = false;
                if (uIndex == CCTOUCHBEGAN)
                {
                    //调用handler的ccTouchBegan方法.
                    //并捕获返回值
                    bClaimed = pHandler->getDelegate()->ccTouchBegan(pTouch, pEvent);

                    if (bClaimed)
                    {
                        //如果handler返回true
                        //把touch对象标记到handler的处理数组中
                        pHandler->getClaimedTouches()->addObject(pTouch);
                    }
                }
                else if (pHandler->getClaimedTouches()->containsObject(pTouch))
                {
                    //该touch对象在handler的处理数据中
                    // moved ended canceled
                    bClaimed = true;
                    //根据不同状态,调用handler的不同函数
                    switch (sHelper.m_type)
                    {
                    case CCTOUCHMOVED:
                        pHandler->getDelegate()->ccTouchMoved(pTouch, pEvent);
                        break;
                    case CCTOUCHENDED:
                        pHandler->getDelegate()->ccTouchEnded(pTouch, pEvent);
                        pHandler->getClaimedTouches()->removeObject(pTouch);
                        break;
                    case CCTOUCHCANCELLED:
                        pHandler->getDelegate()->ccTouchCancelled(pTouch, pEvent);
                        pHandler->getClaimedTouches()->removeObject(pTouch);
                        break;
                    }
                }

                //这里涉及到是否处理下一个handler
                //一定是同时两个条件满足才会停止处理下一个
                //bClaimed 被handler处理了
                //isSwallowsTouches 是swallows的,也可以理解为handler是阻挡touch继续传递的
                //isSwallowsTouches 是怎么设置的呢,稍候会说
                if (bClaimed && pHandler->isSwallowsTouches())
                {
                    if (bNeedsMutableSet)
                    {
                        pMutableTouches->removeObject(pTouch);
                    }

                    break;
                }
            }
        }
    }

    //
    // process standard handlers 2nd
    //

    //这里是处理多点触控了,原理差不多
    if (uStandardHandlersCount > 0 && pMutableTouches->count() > 0)
    {
        CCStandardTouchHandler *pHandler = NULL;
        CCObject* pObj = NULL;
        CCARRAY_FOREACH(m_pStandardHandlers, pObj)
        {
            pHandler = (CCStandardTouchHandler*)(pObj);

            if (! pHandler)
            {
                break;
            }

            switch (sHelper.m_type)
            {
            case CCTOUCHBEGAN:
                pHandler->getDelegate()->ccTouchesBegan(pMutableTouches, pEvent);
                break;
            case CCTOUCHMOVED:
                pHandler->getDelegate()->ccTouchesMoved(pMutableTouches, pEvent);
                break;
            case CCTOUCHENDED:
                pHandler->getDelegate()->ccTouchesEnded(pMutableTouches, pEvent);
                break;
            case CCTOUCHCANCELLED:
                pHandler->getDelegate()->ccTouchesCancelled(pMutableTouches, pEvent);
                break;
            }
        }
    }

    if (bNeedsMutableSet)
    {
        pMutableTouches->release();
    }

    //
    // Optimization. To prevent a [handlers copy] which is expensive
    // the add/removes/quit is done after the iterations
    //
    m_bLocked = false;

    //这里是处理touch期间不能添加和删除handler的地方了
    //touch期间handler的rm和add都会放在一个数组里,这里touch处理完了
    //就开始处理这些数组了
    if (m_bToRemove)
    {
        m_bToRemove = false;
        for (unsigned int i = 0; i < m_pHandlersToRemove->num; ++i)
        {
            forceRemoveDelegate((CCTouchDelegate*)m_pHandlersToRemove->arr[i]);
        }
        ccCArrayRemoveAllValues(m_pHandlersToRemove);
    }

    if (m_bToAdd)
    {
        m_bToAdd = false;
        CCTouchHandler* pHandler = NULL;
        CCObject* pObj = NULL;
        CCARRAY_FOREACH(m_pHandlersToAdd, pObj)
         {
             pHandler = (CCTouchHandler*)pObj;
            if (! pHandler)
            {
                break;
            }

            if (dynamic_cast<CCTargetedTouchHandler*>(pHandler) != NULL)
            {
                forceAddHandler(pHandler, m_pTargetedHandlers);
            }
            else
            {
                forceAddHandler(pHandler, m_pStandardHandlers);
            }
         }

         m_pHandlersToAdd->removeAllObjects();
    }

    if (m_bToQuit)
    {
        m_bToQuit = false;
        forceRemoveAllDelegates();
    }
}
```

我们预留了一些问题:

+ touch的优先级是怎么确定的?
+ 一个handler被处理之后是不是其他的handler都无法处理了?
+ 注册touchhandler的时候基本都是在调用这个函数

```c++
//pDelegate 不用说了,注册的处理对象
//nPriority 这就是处理的优先级了,数字越小,优先级越高
//看以参看forceAddHandler函数
//bSwallowsTouches 这个参数就代表这个handler是否会吞掉touch
//是的话,就不会处理优先级比他低的handler了
//否的话,就会继续处理优先级低的handler
//这就是为什么scrollview处理之后,scrollview的容器Layer,即使优先级更低
//也能接收到touch事件了,因为scrollview是不吞掉touch的

void CCTouchDispatcher::addTargetedDelegate(CCTouchDelegate *pDelegate, int nPriority, bool bSwallowsTouches)
{
    CCTouchHandler *pHandler = CCTargetedTouchHandler::handlerWithDelegate(pDelegate, nPriority, bSwallowsTouches);
    if (! m_bLocked)
    {
        forceAddHandler(pHandler, m_pTargetedHandlers);
    }
    else
    {
        /* If pHandler is contained in m_pHandlersToRemove, if so remove it from m_pHandlersToRemove and return.
         * Refer issue #752(cocos2d-x)
         */
        if (ccCArrayContainsValue(m_pHandlersToRemove, pDelegate))
        {
            ccCArrayRemoveValue(m_pHandlersToRemove, pDelegate);
            return;
        }

        m_pHandlersToAdd->addObject(pHandler);
        m_bToAdd = true;
    }
}
```
