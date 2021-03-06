# Questions
- How can a recipe system work with complex layouts?
- How can axes be reused in recipes to plot a second layer on top of a previous?
- How are legends constructed in single recipes and how are they correctly updated in case of a second layer plotted on an existing one?
- What should be sensible defaults for plotting with base commands, what should they return?

With MakieLayout, high-level elements are treated as mostly autonomous containers
that take care of how their content is presented. They adjust their visual size and position
by receiving a suggested bounding box. This should usually (but doesn't need to)
be generated by a GridLayout that the element is assigned to. Therefore, to be placed
in a scene, an element does not need to be in a layout, but being in a layout moves
and resizes the element so it sensibly fits in with the other content. Usually, most
elements will exist in a layout, so incorporating layouts into the Makie pipeline
should be the default mode of thinking.

GridLayouts are flexibly resizable, columns and rows can be added and removed,
and they can be placed inside other GridLayouts. Therefore, a whole group of
elements can be placed inside an existing layout as a cohesive layout element using
a common GridLayout. This makes it possible to easily assemble larger layouts from
smaller parts. A subplot group can be generated in a separate function (or recipe)
and the result be attached to an existing Scene. This way, complex plots can be assembled
easily.

There are multiple important objects that the user should keep track of in order
to correctly build their visualization and update parameters if they wish. These are:

- The top-level Scene in which all other content is placed
- The layout tree in form of the top GridLayout, or even every single used GridLayout
- Larger content elements of the top-level Scene such as other Scenes or SceneLikes
(LAxis is a SceneLike) or elements such as LText, LColorbar, LLegend, etc.
- Plots that have been added to Scenes or LAxis instances

## Plotting without a scene, layout or axis

The user will not always want to specify all parameters needed for a single correctly
plotted element. For example, a Scene should be generated when creating a Scatter
plot, if there is no Scene given as a mutated argument. In the same way, an LAxis
could be needed if none exists yet, which would be placed in the Scene. And for
the LAxis to be placed correctly, a layout will be needed, too. Of course, the Scatter
object itself should also be accessible as a result of the function call.
Therefore, a simple Scatter plot can result in four (or even more?) objects being created.

```julia
scene, layout, ax, scat = scatter(rand(10, 2))
```

It is probably not desirable to force the user to assign all of these return objects to variables
every time they create a simple plot. Single values could easily be retrieved using
dot access, if they are needed.

```julia
scene = scatter(rand(10, 2)).scene
layout = scatter(rand(10, 2)).layout
```

It should still be possible to display the scene after a single command like this
is run, even though the return type is not Scene anymore. This could be achieved with
multiple dispatch on any NamedTuple that partially matches our return type signature.

```julia
function scatter(x, y)
    # create scene, layout, ax and scat
    # then return a NamedTuple
    NamedTuple(scene = scene, layout = layout, axis = ax, plot = scat)
end

function Base.display(nt::NamedTuple{T, Tuple{Scene,GridLayout,U, V}}) where {T, U, V}
    display(nt.scene)
end
```

## Plotting into an existing scene and layout

In this case an existing scene is used and an object added to it. There could already
be a top-level layout associated with the scene, in this case it should be used
to add the new element to it. A new axis could be created, in this case it should
be returned as well. In general, it should be clear from the inputs what outputs
are expected, not so much because of the performance hit of type instabilities, but
so that the user knows a priori how to deconstruct the returned NamedTuple

```julia
# layout[1, 1] should construct an object like `GridPosition(layout, 1, 1)` so it
# can be dispatched on to put the resulting object into the specified layout position.
axis, plotobj = plot!(scene, layout[1, 1], xyz)
```
> ⚠️ This could be a problem if the user doesn't want to plot into an axis. Maybe for this purpose the `raw` keyword could still be used.

## Plotting into an existing axis

In this case the scene and layout are already decided through the existing axis,
so only the plot object needs to be returned.

```julia
plotobj = plot!(ax, xyz)
```

# Recipes

## Recipes that create layouts

It should be straightforward to allow recipes to place their created content
in an existing layout. For this, again a new type `GridPosition` could be used to
dispatch on, that specifies layout and position at the same time.

```julia
somerecipe!(scene, layout[1, 1], xyz)
```

If the GridLayout type was amended so it can have an optional parent Scene then
this could simplify plotting syntax, as any layout in a layout tree whose parent
has a Scene attached could serve as a surrogate for that Scene.

```julia
# if this holds
parent(layout) == scene

# then these two things should be equivalent
axis, plotobj = plot!(scene, layout[1, 1], xyz)
axis, plotobj = plot!(layout[1, 1], xyz)
```

The `axis` in the previous two function calls would be created because there was
no specific axis given to the function. This could potentially be confusing behavior
as users might expect to be able to just plot into a layout position and thereby
target the axis already located there.

Plotting like in the above example could also return new layout object if we're dealing
with a recipe that creates its own sublayout. To have a clear expectation whether a
recipe call creates a sublayout or not, a trait could be used (for example `CreatesSublayout`).





# Function signatures:

## Non-layout creating

```julia
scene, layout, axis, plotobj = plot(args...)
       layout, axis, plotobj = plot!(scene, args...)
               axis, plotobj = plot!(scene, layout[1, 1], args...)
               axis, plotobj = plot!(layout[1, 1], args...) # if layout has scene connected
                     plotobj = plot!(axis, args...)
```

## Layout creating

```julia
scene, layout, plotlayout, plotaxes, plotobjects = plot(args...)
       layout, plotlayout, plotaxes, plotobjects = plot!(scene, args...)
               plotlayout, plotaxes, plotobjects = plot!(scene, layout[1, 1], args...)
               plotlayout, plotaxes, plotobjects = plot!(layout[1, 1], args...) # if layout has scene connected
                                     plotobjects = plot!(plotaxes, args...)
```

Maybe the first version should also create a sublayout so the two signature groups collapse into one:

```julia
scene, layout, axislayout, axis, plotobj = plot(args...)
       layout, axislayout, axis, plotobj = plot!(scene, args...)
               axislayout, axis, plotobj = plot!(scene, layout[1, 1], args...)
               axislayout, axis, plotobj = plot!(layout[1, 1], args...) # if layout has scene connected
                                 plotobj = plot!(axis, args...)
```

Bundle `axislayout, axis` into one object? It seems to always go together in the signatures.
This is kind of the "infrastructure" that a plot brings with it.

```julia
scene, layout, infrastructure, plotobj = plot(args...)
       layout, infrastructure, plotobj = plot!(scene, args...)
               infrastructure, plotobj = plot!(scene, layout[1, 1], args...)
               infrastructure, plotobj = plot!(layout[1, 1], args...) # if layout has scene connected
                               plotobj = plot!(infrastructure, args...)
```

What type should `infrastructure` have? Flexible enough for any plot type, also reuseable between different plot types, like adding a `Scatter` to infrastructure returned from `Lines`.
Maybe just NamedTuple with a convention for fields.
This convention could be tied to Traits like `SingleAxis`, and `FacetedAxes` or `CustomAxes`

Fields for `SingleAxis`
- layout
- axis
- support

Fields for `FacetedAxes`
- layout
- axes # 2D array
- support

Fields for `CustomAxes`
- layout
- axes # named tuple
- support


# Keyword arguments

There needs to be a robust mechanism to get keyword arguments to where they should go from a very high-level call
like `plot(args...; kwargs...)`.
Maybe this can be achieved by defining a few high-level keywords that other recipes can't use like `scene` or `axis` / `axes` that route their named-tuple arguments separately from the rest.
Similar to Makie's nested attributes?
