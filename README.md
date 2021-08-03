# JavaGl.Obj
[![Build](https://github.com/Saratsin/JavaGl.Obj/actions/workflows/build_and_nuget_publish.yml/badge.svg)](https://github.com/Saratsin/JavaGl.Obj/actions/workflows/build_and_nuget_publish.yml) 
[![Nuget](https://img.shields.io/nuget/v/JavaGl.Obj)](https://www.nuget.org/packages/JavaGl.Obj)

Xamarin.Android bindings library for the [de.javagl:obj](https://github.com/javagl/Obj) library.  
Simple Wavefront OBJ files loader and writer.

# Java Samples

Java Samples showing how to use this library are available in the [ObjSamples project](https://github.com/javagl/ObjSamples).
    

# Overview

This is a simple loader and writer for Wavefront `.OBJ` files. The elements
that are currently supported are

 - Vertices
 - Texture coordinates
 - Normals
 - Faces (with positive or negative indices)
 - Groups
 - Material groups
 - MTL files
 
The `IObj` interface is basically an in-memory representation of an OBJ file.
It combines a `IReadableObj`, which provides the contents of the OBJ file,
and a `IWritableObj`, which may receive elements like vertices and faces
in order to build an OBJ in memory.

The `ObjReader` class may either create a new `Obj` object directly 
from an input stream, or pass the elements that are read from the input 
stream to a `IWritableObj`.

The `ObjWriter` class offers a method to write a `IReadableObj` object
to an output stream.

The `ObjData` class offers various methods to obtain the data that is
stored in a `IReadableObj` as plain arrays or *direct* buffers. 

The `ObjUtils` class offers basic utility methods for general operations
on the OBJ data. 

## Rendering OBJ data with OpenGL
   
The `ObjUtils` class contains methods that aim at preparing the OBJ so 
that it may easily be rendered with OpenGL. These methods may...

 - convert a single group of an OBJ into a new OBJ
 - triangulate OBJ data
 - make sure that texture coordinates or or normal coordinates are unique
   for each vertex
 - convert an OBJ to an OBJ that is uses the same index sets for vertices,
   texture coordinates and normals

The latter operations are also summarized in one dedicated method, namely
the `ObjUtils.ConvertToRenderable` method:

    var inputStream = ...;

    var obj = ObjUtils.ConvertToRenderable(ObjReader.Read(inputStream));

    var indices = ObjData.GetFaceVertexIndices(obj);
    var vertices = ObjData.GetVertices(obj);
    var texCoords = ObjData.GetTexCoords(obj, 2);
    var normals = ObjData.GetNormals(obj);

These buffers may directly be used as the data for vertex buffer objects (VBO)
in OpenGL. 

### Extracting material groups

An OBJ may contain multiple material definitions. When such an OBJ should
be rendered with OpenGL, this usually means that there will be one shader
for each material - or at least, different textures may have to be used
for different parts of the objects. This library offers methods to extract 
the parts of the OBJ that have the same material. In the OBJ format, these 
groups consist of the triangles that follow one `usemtl` directive.

When such an OBJ file is read, the resulting material groups may be obtained
from the `IReadableObj` object, and each of them can be converted into a new
`IObj` object using the `ObjUtils.GroupToObj` method. 

The `ObjSplitting` class contains a convenience method for this:

    var obj = ObjReader.Read(...);
    var mtlObjs = ObjSplitting.SplitByMaterialGroups(obj);

Each of these `IObj` objects may then be converted into a renderable OBJ,
using the `ObjUtils.ConvertToRenderable` method as described above, 
and then be rendered with the appropriate shader for the respective
material.

### Limiting the number of vertices per OBJ

In certain environments, the number of vertices that may be involved in
one rendering call is limited. Particularly, in WebGL or OpenGL ES 2.0,
the indices that are used for indexed draw calls may only be of the type
`GlUnsignedShort`, which means that no object may have more than
65k vertices. In these cases, larger OBJ files have to be split into
multiple parts. Additionally, the index buffers that are passed to 
the rendering API may not contain (4-byte) `int` elements, but only
(2-byte) `short` elements. 

The `ObjSplitting` class contains a method that allows splitting an
OBJ into multiple parts, each having only a maximum number of vertices.
Additionally, the `ObjData` class contains methods for converting 
an `IntBuffer` into a `ShortBuffer`. 
 
So in order to split a large OBJ into multiple parts, and render each
part with WebGL or OpenGL ES 2.0, the following code can be used:

    var largeObj = ObjReader.Read(...);
    var renderableObj = ObjUtils.ConvertToRenderable(largeObj);
    
    if (renderableObj.NumVertices > 65000)
    {
        // If this has to be rendered with OpenGL ES 2.0, then
        // the object may not contain more than 65k vertices!
        // Split it into multiple parts: 
        var renderableParts = ObjSplitting.SplitByMaxNumVertices(renderableObj, 65000);
        foreach (var renderablePart in renderableParts)
        {
            // Obtain the indices as a "short" buffer that may
            // be used for OpenGL rendering with the index 
            // type GlUnsignedShort
            var indices = ObjData.ConvertToShortBuffer(ObjData.GetFaceVertexIndices(renderablePart));
            ...
            
            SendToRenderer(indices, ...);
        }
     }
     ...
