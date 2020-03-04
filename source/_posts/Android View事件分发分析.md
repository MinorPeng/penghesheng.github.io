---
title: Android View事件分发分析
tag: Android源码
category: Android

---

<meta name="referrer" content="no-referrer" />



# View事件分发源码

这次没有脑图，因为涉及不到很多方法，因为就那么几个方法

## 重要方法

事件分发的三个重要方法：

1. `public boolean dispatchTouchEvent(MotionEvent event)`
    用来进行事件的分发，如果事件能够传到当前View，此方法一定调用，返回结果受当前View的onTouchEvent和下级View（子View）的dispatchTouchEvent方法的影响
2. `public boolean onInterceptTouchEvent(MotionEvent event)`
    在dispatchTouchEvent方法的内部调用，用来判断是否拦截这个事件，如果当前View拦截了这个事件，那么在同一个事件序列中，此方法不会再被调用。返回结果表示是否拦截当前事件
3. `public boolean onTouchEvent(MotionEvent event)`
    在dispatchTouchEvent方法中调用，用来处理点击事件。返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件

## 事件分发

对于一个点击事件来说，传递顺序是Activity->Window->View，也就是事件总是先传给Activity，然后给Window，最后到达View，Activity和Window都是默认不处理的，到达View后，先是根View，再是ViewGroup、子View，当前面都不拦截处理，最后会到达最后一个子View，如果这个子View不消耗这个事件，事件又会往回传，最终又回到Activity。

先看**Activity中的dispatchTouchEvent方法**

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    //将事件默认交付给所附属的Window进行分发，如果返回true，事件循环结束
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    //附属Window的dispatch方法返回false（事件没有被处理），则事件交由Activity的onTouchEvent方法处理，Window时不会处理这个事件的，只会传递
    return onTouchEvent(ev);
}

public boolean onTouchEvent(MotionEvent event) {
    if (mWindow.shouldCloseOnTouch(this, event)) {
        finish();
        return true;
    }
    return false;
}
```

Activity默认调用了所附属Window的Dispatch方法，由于Window只是个抽象类，所以这里调用的也就是个抽象方法`public abstract boolean superDispatchTouchEvent(MotionEvent event);`，所以看Window的唯一实现类PhoneWindow（原因如下）

> The only existing implementation of this abstract class is
> android.view.PhoneWindow, which you should instantiate when needing a
> Window.

所以接着看PhoneWindow中的superDispatchTouchEvent方法

```java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;
...

@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}
```

Window并没有做什么事情，就直接进行了事件的分发，通过源码知道mDecor是一个DecorView对象，而且还是这个Window最顶层的View

接着看DecorView中的superDispatchTouchEvent方法

```java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
}
```

DecorView也是什么都不做，直接调用了父类的dispatchTouchEvent方法，那么DecorView的父类到底是什么？

`public class DecorView extends FrameLayout implements RootViewSurfaceTaker, WindowCallbacks`
通过源码知道，DecorView是一个继承FrameLayout，FrameLayout就是一个View的子类，所以事件最后到了View。其实DecorView是Window的顶层View，这个View就是我们系统为Window内置的View，而我们在Activity中通过setContentView方法设置的Activity的布局就是DecorView的一个子View。

言归正传，DecorView中调用了父类的dispatchTouchEvent方法，那么我们接着看它父类中的dispatch方法。

DecorView是继承FrameLayout的，到FrameLayout中发现，这里面并没有dispatchTouchEvent方法，那么DecorView是调用的哪里的呢？其实FrameLayout这个View是继承于ViewGroup的，在FrameLayout中并没有重写这个dispatchTouchEvent方法，所以DecorView调用父类的dispatchTouchEvent方式其实就是ViewGroup中的

那好，接着看这个ViewGroup中的dispatchTouchEvent方法

```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    //这个成员变量在ViewGroup中没找到定义，最后看到View源码时，其实是来自View的
    //mInputEventConsistencyVerifier是否为null，通过查找这个类，它其实是用来检查事件是否是通过输入传进的，这个不是View事件分发的重点，就先不管了
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }

    // If the event targets the accessibility focused view and this is it, start
    // normal event dispatch. Maybe a descendant is what will handle the click.
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }

    //默认返回值是false
    boolean handled = false;
    //这里也是个检查，一般返回的是true
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            //重置FLAG_DISALLOW_INTERCEPT这个标记位
            resetTouchState();
        }

        //检查是否拦截
        // Check for interception.
        final boolean intercepted;
        //事件类型为ACTION_DOWN或者mFirstTouchTarget不为null执行，
        //API27，2617行处mFirstTouchTarget进行了赋值操作
        //从赋值那里可以看出，当事件由ViewGroup的子View成功处理时，mFirstTouchTarget就会被赋值并指向子View，就是说当ViewGroup不拦截事件并交由子view处理，mFirstTouchTarget就不为null，反之就为null
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            //判断ViewGroup自己是否需要拦截这个事件
            if (!disallowIntercept) {
                //需要拦截时，调用自己的onInterceptTouchEvent方法
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        // If intercepted, start normal event dispatch. Also if there is already
        // a view that is handling the gesture, do normal event dispatch.
        //如果要拦截，且有正常的处理，那就处理了吧（自己翻译吧）
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        //如果事件没有被取消，并且ViewGroup也没有拦截
        if (!canceled && !intercepted) {

            // If the event is targeting accessiiblity focus we give it to the
            // view that has accessibility focus and if it does not handle it
            // we clear the flag and dispatch the event to all children as usual.
            // We are looking up the accessibility focused host to avoid keeping
            // state since these events are very rare.
            //这个注释可以翻译翻译，当获取焦点的view不处理这个事件时，就分发给所有的子View
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                    ? findChildWithAccessibilityFocus() : null;

            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                        : TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                //清除之前的事件
                removePointersFromTouchTargets(idBitsToAssign);
                //获取子View数量
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    //选择那些可以收到这个事件的View
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    //遍历所有子View，进行分发
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        //子View的下标，在有事件之前，ViewGroup已经有了所有的子View的信息（不然怎么绘制呢）
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        // If there is a view that has accessibility focus we want it
                        // to get the event first and if not handled we will perform a
                        // normal dispatch. We may do a double iteration but this is
                        // safer given the timeframe.
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        //子View是否能够接收事件，判断子View是否在播动画，点击事件是否在子View区域内
                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        //进行具体的分发，后面会看看这个方法，当子View确定要接收处理这个事件时，会返回true，这样事件就交给这个子View处理了并且跳出遍历子View这个循环；如果返回false，就继续下一个循环，分发给下一个子View
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            mLastTouchDownTime = ev.getDownTime();
                            if (preorderedList != null) {
                                // childIndex points into presorted list, find original index
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            //在addTouchTarget对mFirstTouchTarget的赋值，前面有提到判断它是否位null的哦
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }

                        // The accessibility focus didn't handle the event, so clear
                        // the flag and do a normal dispatch to all children.
                        ev.setTargetAccessibilityFocus(false);
                    }
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            //当没有子View进行分发的时候，注意传入child的参数为null
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            //后续同一系列的事件都交由这一个子View处理，或者要取消事件的时候
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    return handled;
}
```

ViewGroup中这个方法代码很多（我也不知道有多少行），不要慌，慢慢分析

首先，我们先看ViewGroup到达是什么
`public abstract class ViewGroup extends View implements ViewParent, ViewManager`
毫无疑问，这是继承子View的一个类，而且是一个抽象类，通过名字就大概可以猜出这个类是干嘛的了，就是一个View组，包含View的。通俗的来说，View嵌套View的时候，最外层的那个View就可以叫ViewGroup。好了，接下来好好分析这个ViewGroup中重写View的dispatchTouchEvent方法吧

注意看注释哦！！！

首先我们看`if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null)`这个判断，当事件类型为ACTION_DOWN或者mFirstTouchTarget不为null执行（mFirstTouchTarget是什么，看注释哦），那么若是事件为MOVE或UP的时候，这个条件判断就是false，将导致ViewGroup的onTnterceptTouchEvent不会再被调用，且同一序列中的其他事件都会默认交给它处理。还有一种特殊情况就是FLAG_DISALLOW_INTERCEPT标记位，通过requestDisallowInterceptTouchEvent方法设置，一般用于子View中。这个标记位一旦设置后，ViewGroup将无法拦截处理ACTION_DOWN以外的其他点击事件。（ViewGroup在分发事件时，如果事件是ACTION_DOWN就会重置这个标记位，将导致子View中设置的这个标记位无效，因此在面对DOWN事件，ViewGroup总会调用onInterceptTouchEvent来询问自己是否要拦截）这也就意味着ViewGroup的onInterceptTouchEvent不是总是会调用。

在注释中我们提高过mFirstTouchTarget是通过内部的addTouchTarget方法进行赋值的，我们先看看这个函数

```java
private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
    //mFirstTouchTarget其实是一种单链表结构，是否被赋值影响到ViewGroup对事件的拦截策略
    final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
}
```

好了，接着看重点提到的分发给子View，不管是分发给了子View，还是没有子View接收，都是调用了dispatchTransformedTouchEvent方法，那就看看这个私有方法

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    //取消事件的处理
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }

    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }

    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);

                handled = child.dispatchTouchEvent(event);

                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }

    // Perform any necessary transformations and dispatch.
    //真正分发的地方
    if (child == null) {
        //没有子View进行分发时，就会直接调用ViewGroup的父类的dispatchTouchEvent方法（也就是View）
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        //有子View，就分发给子View
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    // Done.
    transformedEvent.recycle();
    return handled;
}
```

直接看到方法的最后，对child进行了是否为null的判断，这个值就决定了是分发给子View还是ViewGroup自己处理这个事件，通过前面的分析，我们知道这两种情况就是分发给子View和ViewGroup没有子View的情况。当ViewGroup没有子View进行分发时，传进的child是null值，然后就调用了自己父类View的dispatchTouchEvent方法；有子View的时候就进行了子View的分发，调用子View的dispatchTouchEvent

其实不管有没有子View，最后都调用了View的dispatchTouchEvent方法（因为有子View进行分发的时候，就交给子View进行分发，然后循环调用，最后一个子View时也就没有了子View），那么就看看View的dispatchTouchEvent方法

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...

    if (onFilterTouchEventForSecurity(event)) {
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        //noinspection SimplifiableIfStatement
        //ListenerInfo就是那些onClik、onLongClick，等会可以看看
        ListenerInfo li = mListenerInfo;
        //检查ListenerInfo、其中的OnTouchListener是否为null，onTouchListener是否返回true
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        //只有当OnTouchListener返回false才会调用onTouchEvent方法
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }

    // Clean up after nested scrolls if this is the end of a gesture;
    // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
    // of the gesture.
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }

    return result;
}
```

这个方法虽然看着代码多，但是逻辑很简单，因为到了这里就是纯粹的View，就没ViewGroup那么多事了，它无法再往下传递事件，就只有处不处理事件

首先判断ListenerInfo是否为null、有没有设置OnTouchListener，如果OnTouchListenter中的onTouch方法返回true，那么onTouchEvent就不会被调用（说明OnTouchListener优先级高于onTouchEvent），ListenerInfo就是很多监听接口集合类，它的源码如下：

```java
static class ListenerInfo {
    protected OnFocusChangeListener mOnFocusChangeListener;

    private ArrayList<OnLayoutChangeListener> mOnLayoutChangeListeners;

    protected OnScrollChangeListener mOnScrollChangeListener;

    private CopyOnWriteArrayList<OnAttachStateChangeListener> mOnAttachStateChangeListeners;

    public OnClickListener mOnClickListener;

    protected OnLongClickListener mOnLongClickListener;

    protected OnContextClickListener mOnContextClickListener;

    protected OnCreateContextMenuListener mOnCreateContextMenuListener;

    private OnKeyListener mOnKeyListener;

    private OnTouchListener mOnTouchListener;

    private OnHoverListener mOnHoverListener;

    private OnGenericMotionListener mOnGenericMotionListener;

    private OnDragListener mOnDragListener;

    private OnSystemUiVisibilityChangeListener mOnSystemUiVisibilityChangeListener;

    OnApplyWindowInsetsListener mOnApplyWindowInsetsListener;

    OnCapturedPointerListener mOnCapturedPointerListener;
}
```

其中就有OnTouchListener、OnClickListener、OnLongClickListener这些常用的监听接口

接着看onTouchEvent方法

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();

    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    //这是View不可用状态下的点击事件
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // A disabled view that is clickable still consumes the touch
        // events, it just doesn't respond to them.
        return clickable;
    }
    //如果设置有代理，会调用mTouchDelegate.onTouchEvent(event)
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    //clickable为true就会消耗事件了
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        //具体处理
        switch (action) {
            case MotionEvent.ACTION_UP:
                ...
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    ....

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                //UP的时候出发performClick方法，如果设置了OnClickListener，内部就会调用onClick方法
                                performClick();
                            }
                        }
                    }
                    ...
                }
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                //DOWN事件处理
                ...
                break;

            case MotionEvent.ACTION_CANCEL:
                //取消事件的处理
                ...
                break;

            case MotionEvent.ACTION_MOVE:
                //MOVE事件处理
                ...
                break;
        }

        return true;
    }

    return false;
}
```

首先是判断View是否处于不可用的状态（比如disable），这种情况下照样会消耗点击事件，尽管看起来不可用。接着看View是否有代理。然后就是onTouchEvent的具体处理了，当clickable为true和(viewFlags & TOOLTIP) == TOOLTIP的时候，就可以消耗事件了。其中clickable是这样定义的`final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;`也就是当View的CLICKABLE、LONG_CLICKABLE、CONTEXT_CLICKABLE为true就可以消耗了（这些可以通过setClickable、setLongClickable来设置），而TOOLTIP是表示事件的显示的（悬浮、长停）

当ACTION_UP发生时，会出发performClick方法

```java
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        //当设置了OnClickListener时，对调用其onClick方法
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

    notifyEnterOrExitForAutoFillIfNeeded(true);

    return result;
}
```

好了，这就是点击事件分发的整个流程了。其实点击事件分发没有什么难的，就是一层一层往下传递，交给下面的，如果处理不了再返回来，整个传递过程Window只是参数传递，不会处理，而Activity在View不处理的时候会自己处理（重写onTouchEvent方法），而到了View，先是根View（DecorView)是一个ViewGroup，你自己的通过setContentView设置的布局只是它的一个子View（但也是个ViewGroup），所以从DecorView开始，就先是ViewGroup的事件分发，然后子View进行分发，一直到最后一个子View不能进行分发为止

## 感谢任玉刚大神的**Android开发艺术探索**