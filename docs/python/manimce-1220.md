# Add sanity checks to :meth:`~.Mobject.add_to_back`
[ManimCommunity/manim#1220](https://github.com/ManimCommunity/manim/pull/1220)

## Issue
`Mobject.add()` contains checks to make sure that:
- `Mobject` doesn't contain itself
- The object added is of type `Mobject`
- No submobject gets added twice

`Mobject.add_to_back()` doesn't have these checks.

## Planning and Implementation
**Resources and Annotations:**
- https://github.com/ManimCommunity/manim/blob/master/manim/mobject/mobject.py - `Mobject.add()` implementation, for reference
- https://github.com/ManimCommunity/manim/blob/master/tests/test_vectorized_mobject.py - `Mobject` tests, for reference

Since `Mobject.add_to_back()` does not have the necessary checks in place like `Mobject.add()` does, we can closely follow the implementation and raise exceptions as needed.

Using existing code as reference:
```py
# manim/mobject/mobject.py

class Mobject(Container):
    ...
    def add(self, *mobjects: "Mobject") -> "Mobject":
        ...
        if config.renderer == "opengl":
            if self in mobjects:
                raise Exception("Mobject cannot contain self")
            for mobject in mobjects:
                if mobject not in self.submobjects:
                    self.submobjects.append(mobject)
                if self not in mobject.parents:
                    mobject.parents.append(self)
            self.assemble_family()
            return self
        else:
            for m in mobjects:
                if not isinstance(m, Mobject):
                    raise TypeError("All submobjects must be of type Mobject")
                if m is self:
                    raise ValueError("Mobject cannot contain self")
            self.submobjects = list_update(self.submobjects, mobjects)
            return self

    ...

    def add_to_back(self, *mobjects: "Mobject") -> "Mobject":
        ...
        # Make sure Mobject does not contain self
        # Raise exception if it does
        
        # Make sure mobject is type Mobject
        # Raise exception if mobject is not Mobject

        # Make sure no submobject gets added twice
        # Filter the incoming mobjects before adding to the list
    ...
```

Implementation
```py
class Mobject(Container):
    ...
    def add_to_back(self, *mobjects: "Mobject") -> "Mobject":
        ...
        # Make sure Mobject does not contain self
        if self in mobjects:
            raise ValueError("Mobject cannot contain self")
 
        for mobject in mobjects:
            # Make sure mobject is type Mobject
            if not isinstance(mobject, Mobject):
                raise TypeError("All submobjects must be of type Mobject")
 
        # Make sure no submobject gets added twice
        filtered = list_update(mobjects, self.submobjects)
        self.remove(*mobjects)
        self.submobjects = list(filtered) + self.submobjects
        return self
    
    ...
```

- `list_update` filters mobjects from submobjects like a set; internal method
- `add_to_back` is actually adding to front of the list because the first objects are rendered first, while subsequent items are rendered on top.

## Unit Testing

Using existing code as reference:
```py
# tests/test_vectorized_mobject.py

...

def test_vgroup_add():
    """Test the VGroup add method."""
    obj = VGroup()
    assert len(obj.submobjects) == 0
    obj.add(VMobject())
    assert len(obj.submobjects) == 1
    with pytest.raises(TypeError):
        obj.add(Mobject())
    assert len(obj.submobjects) == 1
    with pytest.raises(TypeError):
        # If only one of the added object is not an instance of VMobject, none of them should be added
        obj.add(VMobject(), Mobject())
    assert len(obj.submobjects) == 1
    with pytest.raises(Exception):  # TODO change this to ValueError once #307 is merged
        # a Mobject cannot contain itself
        obj.add(obj)

def test_vmob_add_to_back():
    # Test: Mobject cannot contain itself

    # Test: All submobjects must be of type Mobject

    # Test: No submobject gets added twice
... 
```

Implementation:
```py
def test_vmob_add_to_back():
    """Test the Mobject add_to_back method."""
    a = VMobject()
    b = Line()
    c = "text"
    with pytest.raises(ValueError):
        # Mobject cannot contain self
        a.add_to_back(a)
    with pytest.raises(TypeError):
        # All submobjects must be of type Mobject
        a.add_to_back(c)

    # No submobject gets added twice
    a.add_to_back(b)
    a.add_to_back(b, b)
    assert len(a.submobjects) == 1
```