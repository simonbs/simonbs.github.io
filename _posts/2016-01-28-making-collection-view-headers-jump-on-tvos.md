---
layout: post
title: Making Collection View Headers Jump on tvOS
tags: development tvos
hide: true
---

Have you seen those sweet collection view headers Apple added everywhere they got a chance to do so in tvOS? Those that move up a little when you focus the cell beneath it?

They look like this.

![Bouncy Headers](/img/posts/making-collection-view-headers-jump-on-tvos/bouncy-headers.gif)

They are pretty cool. And they are easy to replicate.

I've created HeaderAdjustments to do this. The source code is available on GitHub. In this blog post I'll explain how you can use it in your tvOS app.

When you get the hang of it, it's really simple to add headers that adjust to your app. You can grab the very simple sample code at [github.com/simonbs/BouncyHeaders](http://github.com/simonbs/BouncyHeaders).

The post assumes the following:

- You are comfortable with nested collection views on tvOS. You really should be, you are going to use it everywhere when you get your hands dirty with tvOS.
- You separate your collection view data from your view controllers. You do that, right? You don't keep everything in a huge view controller, right? Right?
- That you are comfortable with UICollectionViewFlowLayout.
- That the content you want to display (e.g. movies, books, photos, ...) is represented using an image. That is, your collection view cells should contain an image view and this image view should have `adjustsImageWhenAncestorFocused` enabled.

When building a tvOS app, it's likely that you are going to display content in multiple sections, much like the App Store on tvOS. To do this, Apple suggests using nested collection views. You should be comfortable with this before continuing. If you aren't, you should take a look at [Apples sample code](https://developer.apple.com/library/tvos/samplecode/UICatalogFortvOS/Listings/UIKitCatalog_CollectionViewContainerCell_swift.html#//apple_ref/doc/uid/TP40016433-UIKitCatalog_CollectionViewContainerCell_swift-DontLinkElementID_9).

We'll assume we want to build an app with a view in which we show three sections of content:

- The first section contains photos of London
- The second section contains photos of Paris
- The third section contains photos of Rome

This is exactly what the sample project does: shows three sections, each containing a list of photos.

A sketch of the application can be seen below. The orange boxes represent three different UICollectionViews. Blue boxes represent three different UICollectionReusableViews. This is encapsulated in a UICollectionView with three sections.

![Bouncy Headers Sketch](/img/posts/making-collection-view-headers-jump-on-tvos/bouncy-headers-sketch.png)

Now let's get our hands dirty.

### Innermost collection view cells

The cells in your innermost collection views (i.e. the ones with the *real* content), should conform to the HeaderAdjustmentsCollectionViewCell protocol. This protocol has a single requirement:

{% highlight swift %}
var headerAdjustmentsImageView: UIImageView { get }
{% endhighlight %}

`headerAdjustmentsImageView` should return the image view in your cell, which has `adjustsImageWhenAncestorFocused` enabled. The focused layout frame (`imageView.focusedFrameGuide.layoutFrame`) is used to check for collapsed with the header views in the outermost collection view.

### Outermost collection view data source

The layout of the outermost collection view should be a UICollectionViewFlowLayout. However, the layout of your inner collection views can be whatever you want.

The outermost collection view holds all of our sections. We'll create a class named SectionsData, **make it inherit from HeaderAdjustmentsCollectionViewData** and conform to UICollectionViewDataSource and UICollectionViewDelegateFlowLayout, just like we're used to.
Inheriting from HeaderAdjustmentsCollectionViewData is important. It listens to focus changes and scroll events and adjusts the headers when necessary.

`-numberOfSectionsInCollectionView:` returns the number of sections in our collection view (3), `-collectionView:numberOfItemsInSection:` always returns 1. This gives us the possibility of adding three headers (UICollectionElementKindSectionHeader), one for each section.

Remember to disable focus for your outermost collection views by returning `false` in `collectionView:canFocusItemAtIndexPath:`.

Make SectionsData return your header views as below and configure them like you want to, e.g. show a section title in a UILabel.

### Configuring your header views

It is **important that your header views inherit from HeaderAdjustmentsCollectionViewHeaderView**. They inherit the following two important methods from HeaderAdjustmentsCollectionViewHeaderView:

{% highlight swift %}
public func adjustForRect(rect: CGRect) { }

public func adjustView(view: UIView, withConstraint constraint: NSLayoutConstraint, inRect rect: CGRect, usingOffset offset: CGFloat = -30, defaultOffset: CGFloat = 0, attachment: Attachment? = nil) { }
{% endhighlight %}

Your header view should override `-adjustForRect:` as it is called to give the header view a chance to adjust its subviews. If the header view is supposed to adjust its sub views upward (i.e. there's a focused cell beneath), the `rect` parameter holds the focused frame of the `headerAdjustmentsImageView` relative to the header view. If the header view should reset its subviews, CGRectNull is supplied as argument.

The former method is a helper method which can be used within `-adjustForRect:`. HeaderAdjustments generally makes no assumptions about how you want to adjust your headers when a collapse occur. Maybe you want to change text colors, text fonts, background colors or what do I know. The most common case is that you want to change the vertical adjustment of subviews (e.g. a title label). The former methods makes this a breeze. It takes the following arguments.

- *view*: The UIView you wish to move up or down, e.g. a title label.
- *constraint*: The NSLayoutConstraint on which to perform the adjustment.
- *rect*: The rect supplied to `-adjustForRect:`.
- *offset*: The offset applied to the supplied constraint if the image view in the collection view cell and the *view* collides. Defaults to 30.
- *defaultOffset*: The offset applied to the supplied constraint if the image view in the collection view cell and the *view* **does not** collide. Defaults to 0.
- `attachment`: Whether your view is "attached" to the left or the right side of the header view, or nil if it has no attachment. More on this below.

You may call the method like shown below in order to adjust the title label upwards 30 pixel when a collapse is detected and move it to a 0 pixel offset when there is no collapse. Not that the helper method automatically determines if there's a collapse.

{% highlight swift %}
    adjustView(titleLabel,
                withConstraint: titleVerticalCenterConstraint!,
                inRect: rect,
                usingOffset: 30,
                defaultOffset: 0,
                attachment: .Left)
{% endhighlight %}
                  
What is the attachment good for, you might ask. Well, when scrolling a horizontal collection view you'll sometimes focus a cell that's not within the visible rectangle of the collection view.
For example, when your collection view is scrolled all the way to the right, and you scroll to the left, you may focus a cell that is left of the header view. When focused, the cell will animate in (scroll in) to be within the visible rectangle of the collection view. In this case there's no collision with the cell that's not within the visible rectangle and the header view, but there will soon be (when the scroll finishes). In those cases we want to keep the title adjusted, even if there are no collisions.
You can attach view to the left or the right of the header view. It is very common to attach title labels to the left of your header view.

That's all there is to it. It's relatively simple to configure is sufficiently generic that the header views of your outermost collection view can contain any subviews that can be adjusted. Also, your innermost collection views can have any collection view layout.

### Sample

You can grab the sample code at [github.com/simonbs/BouncyHeaders](http://github.com/simonbs/BouncyHeaders).

You'll find the core of the app in *Source/Pages/Images*. I use a few "utility classes" that allow me to remove a substantial amount of boilerplate code. These are unrelated to HeaderAdjustments but provide for cleaner code. All the utilities are located in *Source/Utilities*.
