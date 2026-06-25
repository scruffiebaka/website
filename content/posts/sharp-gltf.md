---
title: "Model importer using SharpGLTF for OpenTK"
date: 2026-01-13T18:02:20+00:00
draft: false
---

# Model importer using SharpGLTF for OpenTK

Hey! This blog would be how I implemented a model importer using SharpGLTF that imports .glb as raw data that we can then render using OpenTK. I tried to find a beginner guide on this but there isn't a lot of stuff available apart from the examples (which to be honest proved to be of no use for me)

The game engine I've been working on does not have multiple features so we will only be going over the top.

A glTF file is basically a JSON file that has a scene and multiple nodes and meshes inside. The nodes of that file can have different components attached to it (mesh, camera etc). For my game engine, I converted each node into an instance of Gameobject and then added a Mesh and MeshRenderer component to them.

You can refer to https://github.khronos.org/glTF-Tutorials/gltfTutorial/ for a more detailed tutorial/guide on glTF files.

First step is adding SharpGLTF as a dependency. You can grab the [NuGet package](https://www.nuget.org/packages/SharpGLTF.Core) or get it from its [Github repository](https://github.com/vpenades/SharpGLTF).

### Handling the scenes and nodes

To load a model, we can call `ModelRoot.Load(modelPath)` where modelPath is the path of the model.

From here on, based on how your engine is made, you can either recursively go through all the nodes in the scene, or just take all the meshes and render them.
For my engine, I just created a gameobject for each node and added the children based on the nodes.

Our main model begins at a scene, so we assign a variable to it `var  scene  =  model.DefaultScene;` .
Then, we can loop through all of the nodes in the scene:

    foreach (var  node  in  scene.VisualChildren)
    {
	    //Handle nodes
	    ImportNode(node, gameobject);
    }

To handle the nodes, I created another function `static  void  ImportNode(Node  node, Gameobject  parent)` which can handle each node.

To handle each node, what I did was obtain the local transform of the node using `Matrix4x4  local  =  node.LocalMatrix;`.
We can then decompose the matrix to obtain the values:

    Matrix4x4.Decompose(
	    local,
	    out  Vector3  scale,
	    out  Quaternion  rotation,
	    out  Vector3  translation
    );
And then I assign these values to my Gameobject that is created using the node.

I can then recursively run this function in order to obtain every single node.

    foreach (var  child  in  node.VisualChildren)
    	ImportNode(child, go, shader);

If the node has a mesh, I run another function `ImportMesh(node.Mesh, go, shader)` and in the ImportNode method
`if (node.Mesh  !=  null)
	ImportMesh(node.Mesh, go);`

Where `go` is just my gameobject.
For every mesh, I create another gameobject. 

### Importing the mesh

We can obtain the list primitives from the mesh using `mesh.primitives`. We can then loop through the list.
To actually obtain our data, we need to create and use accessors.

    var  positionAccessor  =  primitive.GetVertexAccessor("POSITION");
    var  normalAccessor  =  primitive.GetVertexAccessor("NORMAL");
    var  texAccessor  =  primitive.GetVertexAccessor("TEXCOORD_0");
    var  indexAccessor  =  primitive.GetIndexAccessor();

We then convert those accessors into arrays

    var  positions  =  positionAccessor.AsVector3Array().ToArray();
    var  normals  =  normalAccessor?.AsVector3Array().ToArray();
    var  uvs  =  texAccessor?.AsVector2Array().ToArray();
    var  indices  =  indexAccessor.AsIndicesArray().ToArray(); 

And these arrays are the data you require in order to build the model. It has the vertex information, (positions, normals, uv's) and the indices.
For my engine, I have a vertex struct:

    public  struct  Vertex
    {
    	public  Vector3  Position;
    	public  Vector3  Normal;
    	public  Vector2  TexCoords;
    }

And I use this struct for my MeshRenderer. Since the position's indexes correspond to the normals and uv's, we can simply loop through it.

    for (int  i  =  0; i  <  positions.Length; i++)
    {
    	Vertex  v  =  new  Vertex();
    	v.Position.X  =  positions[i].X;
    	v.Position.Y  =  positions[i].Y;
    	v.Position.Z  =  positions[i].Z;
    	
    	if (normals  !=  null)
    	{
    		v.Normal.X  =  normals[i].X;
    		v.Normal.Y  =  normals[i].Y;
    		v.Normal.Z  =  normals[i].Z;
    	}
    	
    	if (uvs  !=  null)
    	{
    		v.TexCoords.X  =  uvs[i].X;
    		v.TexCoords.Y  =  1.0f  -  uvs[i].Y;
    	}
    	
    	vertices.Add(v);
    }

And then I can just pass on this information onto my mesh component, which constructs the mesh using the vertex information and the indices.

    meshComp.Initialize(vertices, indicesList);


### Materials

Materials have a lot to offer and frankly, I haven't explored too much into it. I simply obtain the base color and the texture and apply it. In the future, I might build more upon it.

We can obtain the material using `primitive.Material`. The material itself has multiple channels that contain values. The one I need for the diffuse texture is called BaseColor, so we obtain that channel and get its texture and color.

    Material?  mat  =  primitive.Material;
    MaterialChannel?  baseColor  =  mat.FindChannel("BaseColor");
    Texture?  texture  =  null;
    if (baseColor  !=  null)
    {
    	texture  =  baseColor.Value.Texture;
    }

To handle images, I use a library called StbImageSharp. It can load images using memory, and that is exactly what we need.

    if (texture  !=  null)
    {
    	byte[] content  =  texture.PrimaryImage.Content.Content.ToArray();
    	baseTex  =  new  Rendering.Texture(
	    	Rendering.Texture.LoadFromMemory(content),
	    	Rendering.TextureType.texture_diffuse
    	);
    } else
    {
    	baseTex  =  new  Rendering.Texture(
	    	Rendering.Texture.LoadFromFile("Resources/Textures/white.png"),
	    	Rendering.TextureType.texture_diffuse
    	);
    }

`Rendering.Texture` is my the class that handles textures, It simple loads the image, binds it to the texturing target, specifies the texture image, generate mipmaps, set filtering etc. It returns the handle which you can then bind to the texture target.

If a texture is not found, I just load a white 1x1 image. 
We can also get the Color of the BaseColor channel using 

    baseColor.Value.Color.X,
    baseColor.Value.Color.Y,
    baseColor.Value.Color.Z)

And then assign it to the material class.

-----

End result:

![showcase](https://i.ibb.co/7xQXLTfj/image-2.png)

Model: [Half-Life - C1a0a by Maxime66410](https://sketchfab.com/3d-models/half-life-c1a0a-f6889ae319324fb39a00ec1bc87d212a)