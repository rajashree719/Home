## NuGet RestoreGraph and Central Transitive Dependencies

### Summary 
The new Central Package Version feature support will allow defining the package versions in a single unified file ([Spec](https://github.com/NuGet/Home/wiki/Centrally-managing-NuGet-package-versions))
There are two requirements worth to be mentioned for the context of this document :
1. The versions defined in the central package version file (Directory.Packages.props) will be respected for transitive dependencies (in additon to the direct dependencies).
2. A package downgrade due to a version defined in the central package version file (Directory.Packages.props) will be treated as error not as warning.  

Below there is a short summary for the current implementation of the GraphNodes and Graph creation. 

### RestoreGraph Nodes (in a nutshell)
The restore graph nodes are of type [GraphNode](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphNode.cs)

A GraphNode in addition to references to its inner nodes and outer node has 
1. A [Disposition](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/Disposition.cs) property. <br> The Disposition can be one of the values: <br> 
    - Rejected
    - Accepted
    - PotentiallyDowngraded
    - Acceptable
    - Cycle 
2. An Item of type [GraphItem <T>](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphItem.cs)
The generic type of the graph item type is usually a [RemoteResolveResult](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphNode.cs). <br>
The [GraphItem <T>](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphItem.cs) contains 
    - The [LibraryIdentity](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.LibraryModel/LibraryIdentity.cs) (basically Package Name and Version) of the current node 
    - The [RemoteResolveResult](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphNode.cs) instance that contains the Dependencies of the current package
    - A flag ```IsCentralTransitive``` that it is true if the node was created due to a transitive central dependency. This flag was introduced as part of the support for CPVM (central package version management)

### Graph Creation (in a nutshell)

#### Stage 1
The Graph is created recursively by the [RemoteDependnecyWalker.CreateGrahNode](https://github.com/NuGet/NuGet.Client/blob/aa29ed4513a1b2f5149a298df1c4a334963a9a0c/src/NuGet.Core/NuGet.DependencyResolver.Core/Remote/RemoteDependencyWalker.cs#L48). <br>
The root node is the project node. <br>
At each node level the dependencies are evaluated for PossibleDowngrades or Cycles. If there are not any(PossibleDowngrades or Cycles) the ``` CreateGraphNode ``` is invoked recursively. <br>The evaluation of PossibleDowngrades or Cycles is done be walking up the graph and finding a possible node dependency with same id/packageName but with version smaller than the current version (PossibleDowngrades) or by finding cycle based on the package name). <br>
Cycle example  
```
A -> B -> A
```
PotentiallyDowngraded example 
```
A -> B -> C -> D 2.0 
   -> D 1.0 
```

In the snapshot above the D 2.0 node will be PotentiallyDowngraded.

At this stage when the graph is created the nodes can be in one of the three ```Disposition```
1. Acceptable
2. PotentiallyDowngraded
3. Cycle

A sample resulted graph could be as below, where all the nodes are Acceptable only the D 2.0 is PotentiallyDowngraded.
```
ProjectP -> A(acceptable) -> B(acceptable) -> C(acceptable) -> D(PotentiallyDowngraded) 2.0 
                           -> D(acceptable) 1.0 
          -> E(acceptable) -> F(acceptable) -> G(acceptable) 
```

#### Stage 2

The graph above wil be analyzed to determine the nodes that will be accepted (
[Analyze](https://github.com/NuGet/NuGet.Client/blob/aa29ed4513a1b2f5149a298df1c4a334963a9a0c/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/GraphOperations.cs#L25) (through [RestoreTargetGraph.Create](https://github.com/NuGet/NuGet.Client/blob/aa29ed4513a1b2f5149a298df1c4a334963a9a0c/src/NuGet.Core/NuGet.Commands/RestoreCommand/ProjectRestoreCommand.cs#L271))).

During this phase the graph is walked recursively multiple times and:
1. The downgrades list is created and the downgraded nodes removed. The downgrades result [DowngradeResult](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.DependencyResolver.Core/GraphModel/DowngradeResult.cs) has a **From** and **To** nodes that reference the nodes that was downgraded in the favor of the node downgraded.
2. The nodes are accepted or rejected. The highest version wins over the ambiguous versions.
3. Conflicts are detected (if any).
4. If any downgrade resulted in its ```DowngradedTo``` to not be accepted the downgraded item is removed from the downgraded list. This prevent eventual ```Rejected``` nodes to be the ```DowngradedTo``` nodes. 

At the end of the Analyze operation the ```Disposition ``` of the graph nodes will be either ```Accepted``` or ```Rejected```. <br>
The resulted [RestoreTargetGraph](https://github.com/NuGet/NuGet.Client/blob/dev/src/NuGet.Core/NuGet.Commands/RestoreCommand/RestoreTargetGraph.cs) it will contain the flatten list of not ```Rejected``` nodes in addition to the list of conflicts and downgrades. 
The flatten list is a HashSet of accepted nodes.  


### Graph Creation for projects with central versions dependencies
In this part it will be explained how the transitive dependencies that have versions in the central package versions management file will be enforced. 

#### Stage 1

As above at the end of this stage the nodes will need to be in one of the states ```Acceptable```, ```PotentiallyDowngraded``` or ```Cycle```.

When the graph is recursively created if any of the current node dependency (a.k.a transitive dependency) has a version defined centrally the node for this dependency will be uplifted and become a inner node for the root (basically will act as direct dependency). 

If the transitive dependency version is higher than the central defined version the node will be marked as ```PotentiallyDowngraded``` same as in the non CPVM case).

Few snapshots below to illustrate this <br>

Let assume the the ProjectP has packages **A**, **D** and **H** as direct dependencies. The purple element (package **B**) is a package with version defined in the central package version management file.

![image](https://user-images.githubusercontent.com/16580006/81230663-c1fabe80-8fa6-11ea-8f4e-c3d0797a0a5f.png)


The packages **A**, **D** and **H** have also dependencies as below

![image](https://user-images.githubusercontent.com/16580006/81231001-56652100-8fa7-11ea-9818-a59cf962e745.png)

During the ***Stage 1***
1. If package **B** ```does not have``` a version declared centrally the graph after stage one will be

![image](https://user-images.githubusercontent.com/16580006/81231445-033f9e00-8fa8-11ea-8014-0b6c74112c71.png)


2. If package **B** ```has``` a version declared centrally the graph after stage one will be

![image](https://user-images.githubusercontent.com/16580006/81231811-a55f8600-8fa8-11ea-864a-0291dea55ff0.png)


#### Stage 2

As mentioned above after stage 2 the nodes will be either ```Rejected``` or ```Accepted```
1. If package **B** ```does not have``` a version declared centrally the graph after stage two will be

![image](https://user-images.githubusercontent.com/16580006/81232056-0e46fe00-8fa9-11ea-85e6-174b4821c94f.png)


<br> 
2. If package **B** ```has``` a version declared centrally the graph after stage two will be

![image](https://user-images.githubusercontent.com/16580006/81231579-3e41d180-8fa8-11ea-9974-fef15eb01c08.png)


**Graph with downgrades**

Lets' assume a project as below (the purple element is a package with version defined in the central package version management file) 

![image](https://user-images.githubusercontent.com/16580006/81233537-b067e580-8fab-11ea-8c37-a53dd710096c.png)

The package **A** has the dependencies 

![image](https://user-images.githubusercontent.com/16580006/81238464-05f5bf80-8fb7-11ea-8e14-41eee9011e0a.png)

For the above project representation:

### Stage 1

![image](https://user-images.githubusercontent.com/16580006/81238598-4fdea580-8fb7-11ea-97fa-140c0b909fa3.png)


### Stage 2

![image](https://user-images.githubusercontent.com/16580006/81233923-94187880-8fac-11ea-97cf-12106042ea8b.png)

***A new Downgraded element will be added to the list of downgrades.***


**Graph with rejected nodes**

Lets' assume a project as below (the purple element, package **B** is a package with version defined in the central package version management file)

![image](https://user-images.githubusercontent.com/16580006/81235617-3f76fc80-8fb0-11ea-9aa3-c38f7cd2521b.png)


The packages **A** and **H** have dependencies like below

![image](https://user-images.githubusercontent.com/16580006/81235776-95e43b00-8fb0-11ea-88a0-487aeee75a19.png)


### Stage 1 

1. If package **B** ```does not have``` a version declared centrally the graph after stage one will be.

![image](https://user-images.githubusercontent.com/16580006/81235839-bf04cb80-8fb0-11ea-8b8c-33bea5fdae6b.png)


2. If package **B** ```does have``` a version declared centrally the graph after stage one will be

![image](https://user-images.githubusercontent.com/16580006/81236945-320f4180-8fb3-11ea-9ba0-8b76dfa848c3.png)

### Stage 2 

1. If package **B** ```does not have``` a version declared centrally the graph after stage two will be.

![image](https://user-images.githubusercontent.com/16580006/81235891-d643b900-8fb0-11ea-99ab-90f2819e66e1.png)


2. If package **B** ```does have``` a version declared centrally the graph after stage two will be.

![image](https://user-images.githubusercontent.com/16580006/81236855-ff654900-8fb2-11ea-8eb1-5a6701a72d92.png)

As mentioned above the ```Rejected``` nodes will not be included in the flatten graph.

The rest of the graph resolution algorithm stays the same as currently, where any uplifted and not-rejected central transitive dependencies will participate as direct inner nodes for the root node.