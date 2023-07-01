+++
title = "2D UI beginner guide. Learn to rotate/translate/scale"
description = "2D UI beginner guide. Learn to rotate/translate/scale"
date = 2022-04-20T09:19:42+00:00
updated = 2020-04-20T09:19:42+00:00
draft = false
template = "blog/page.html"

[taxonomies]
authors = ["Public"]

[extra]
lead = "Post about 2d transformations that i did in my previous work life."
+++

Recently I had pleasure to make UI for standard rotate/scale/translate controls on element with some bounding box. I told myself: great! this will be easy! Lets google it, make some matrices and this will be quickly finished... must say that it took me a bit longer. Internet didn't help me with comprehensive solution or guide, I needed to stitch stuff together, and at the end of my frustration I got inspired to write this.

I will give small intro about rotation and translation because they can be easily implemented and will focus my attention to scaling that made me warm around my hearth (or maybe that was my frustration). At the end you can find TLDR section with aggregated functions that we call.

# Transformations

Transformation for 2D is constituted of tree things: Rotation, Scaling and Translation and all three things can be represented with one 2x3 matrix (but because of conformity that matrix multiplication give us we add some zero padding and use 3x3). When thinking about matrix most of times I see only a black box, nothing more, I know what I can do with them, and avoid manually setting things. Most important things that we need to take care when matrixing is the order on how we do transformation, it matters, `Mfirst*Msecond` is not same as `Msecond*Mfirst` and it depends if we are using row or column major matrices.

Some info on our setup. Our original element (before transformations), has some size `original_size`, it is rectangular shape (or his bounding box is), it lies in first quadrant with starting position at coordinate beginning. And we are using row major order matrices (This means order of multiplication of matrices is from left to right `Mfirst*Msecond`) additionally, we are using these points:

* `ref_point`: point from where transformation is started for rotation/scale this is corner that we selected, for translation this is point where user clicked and started to drag our element.
* `current_point`: current mouse point, it is point where we want to rotate/scale/translate
* `M` : transformation matrix that was already applied on our element.
* `anchor_point`: other side from `ref_point`, (opposite corner or side). Needed for scaling.
* `center_point` : `ref_point-anchor_point`
* `original_size`: Original size of our element (For some systems, element can be normalized to (1,1), but for this example I think it is better to show how we are effected if there is size).

Okay, lets get started from easy to hard:

## Translation

Translation is most simple of them all, it moves point by specified vector, and nothing more. It is used in tandem with rotation or scaling to center already moved/rotated element, but this will be explained in due time. For our UI you take point when mouse is clicked `ref_point`. And take current point where mouse moved `current_point`. get diff and create translation matrix as this `M=M*translation(current_point-ref_point)` and voila, we are done.

## Rotation

Rotation is little bit more complex (it has little bit more to do) but in same rank as translation. We need reference point `ref_point` for selected element. Rotation is usually, not to say always, done around element center, for this we need `center_point`. And lastly we have `current_point`. As you can guest we need to find the angle between these two vectors `x=ref_point-center_point` and `y=current_point-center_point`. After consulting internet we get this equation:`angle = atan2(norm(cross(x,y)), dot(x,y))`. With angle found we can call function for creating matrix, something like `R=rotation(angle)`. Appending `R` to transformation matrix `M` is done with this simple but very used and important trick: We create another matrix of translation from elements center `T=translation(center_point)`, and it's inverse`Tinv = inverse(T)`. We get matrix that we can use to append transformation to already present points `Ra = Tinv*R*T` and final transformation is `M=M*Ra`. Basically (with `Tinv` we just nullify translation, we then rotate our element around center and apply T to put it back into old position).

## Scale

And we come to scaling, it is best part of this post (it has pictures) :D. We will gradually introducing few things that needs to be done in scaling, we will see how we handle rotation, and shift controls (shift is usually used for aspect ration lock).

![Naive Scale](./naive_scale.png)

Nice, lets start with basic example where our element is not rotated or translated and we just want to scale it. We will use `ref_point` (corner or side usually), and its `anchor_point` and of course we will need `current_point` to tell us where we want to scale to. We calculate `diff=current_point-anchor_point`, get scale as `s=scale(diff/element_size)` and we are done, we have scale matrix that can add to our transformation.

Okay, lets now look on example where we want to take top left corner `ref_point` ( you can follow picture below), in that case our `anchor_point` is positioned at bottom and if we want to scale it properly, to top and left. First difference from previous example is that we will need to move our object so that `anchor_point` is in `(0.0)` coordinate! We still need `diff` and we are calculating it same as before, but because now our axis are flipped, this is second difference, we need to reverse sign of `diff_new=Vector(-diff.x,-diff.y)`. Note, reversing `y` is needed for top side `ref_point` and reversing `x` for left side `ref_point`. We get scale as `s=scale(diff_new/element_size)` . And final third difference from previous example is that after all this we need to take translation of anchor `T=translate(anchor_point)`, calculate inverse `Tinv=inverse(T)` and bind it all together (from left to right) `S=T*s*Tin`.

![Scale](./scale.png)

As you can see diff vector is oriented to negative in reference to our axis, this is reason why we need to flip it, if we didn't do flipping you would get small scale when moving away from top left corner.

This will all work just fine if element is not in any way rotated (Yey rotation!), with rotation we are now in bind how to calculate our diff and extract scale information. But, don't despair, we can use same trick as we did with rotation in a way that we will take `current_position` and inverse of current transformation matrix `Minv = inverse(M)` and get `relative_position=Minv*current_position`. Relative position now presents point relative to our **original** element. We get corner of original element as: `original_anchor_point=original_corners[handler_id]` (take care to select correct corner, it is probably jumbled up with rotation, I had something like `handler_id` to help me with that) and do same as we did in our last example, calculate diff as `diff=relative_position-original_corners[handler_id]`, and if needed invert its axis. Calculate scale as `s=scale(diff_new/element_original_size)` and now similarly as previous scale example we need to move our original element to anchor before we do scaling, bear in mind that that translation represent anchor when our element is **not** transformation. We get `T=translate(original_anchor_point)` and its inverse `Tinv` and we get `S=T*s*Tinv`.

That's great, but how to append scale in current matrix, when scale is something that is done before rotation and translation? We could always prepend scale to `M`, and this is only way to properly add scale, mostly because we are using `S*R*T` order. But how to get matrix to apply directly on already transformed points? Hah, just take `Minv = inverse(M)` and get transformation that we can append on present points as `Sa=Minv*S*M`, and final matrix is `M=M*Sa`.

## Shift scale

Shift scale is scaling where aspect ration is not changed. This means that scale on both axis is equal and we need to choose which axis orientation we will take as primary. We could make it simple and depending on which corner_id is selected that take modulo of two and chose x or y scale, this will work but will be unintuitive. For better solution where depending on position of mouse relative to diagonal of element we will get smother transition between x and y orientation. See picture below:

![Naive Scale](./shyft_scale.png)

With transparent colors we can see zones where we want to take only `x` ( blue color) or take only `y` (marked with red). As noticeable our object is in original position that means our `original_points` is calculated same as in example with rotated object. Slope of diagonals that make these zones are calculated from `original_size` with equation `line_slope = original_size.y/original_size.x` . for second diagonal it is enough to just flip sign and we will get second slope. what we want to check is if point is in blue or red space and we can do that following if statement: (for abbreviate: `op` is `original_point` , `ls` is `line_slope` ): `(op.y < ls**op.x && op.y > -ls**op.x) || (op.y > op.x**ls && op.y < -ls**op.x)`, and if this if statement is true do `scale.y=scale.x` if it is false do opposite. And lastly don't forget that when you are overriding one scale to not override its sign, in example from picture we are taking `y` scale and overriding `x` scale but we need to preserve `x` sign to properly scale our element `x=sign(x)*abs(y)`.

## TLDR

Summary of functions that were called throughout the text:

Translation:
```text
M = M*translation(current_point-ref_point)
```

Rotation:
```text
crp  =ref_point - center_point
ccp = current_point - center_point
angle = atan2(norm(cross(crp,ccp)), dot(crp,ccp))
R = rotate(angle)
T = translation(center_point)
Tinv = inverse(T)
Ra = Tinv*R*T	//diff that can be used to apply to already transformed points
M = M*Ra
```

Scale:
```dwda
Minv = inverse(M)
relative_position = Minv * current_position
original_anchor_point = original_corners[handler_id]
diff = relative_position - original_anchor_point
s = scale(diff/element_original_size)
T = translate(original_anchor_point)
Tinv = inverse(T)
S = T*s*Tinv
Sa = Minv*S*M	//diff that can be used to apply to already transformed points
M=M*Sa
```

Shift scale:
```dwda
scale = (x,y)
line_slope = original_size.y/original_size.x
if (op.y < ls**op.x && op.y > -ls**op.x) || (op.y > op.x**ls && op.y < -ls**op.x) {
	scale.y = x=sign(y)*abs(x)
} else {
	scale.x x=sign(x)*abs(y)
}
```