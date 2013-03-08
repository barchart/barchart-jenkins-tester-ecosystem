Project Structure
-----------------

*Legend:*
```
*  - Releasable, has associated Jenkins build job
+  - Submodule releasable only with parent, no associated Jenkins build job
X  - Not releasable, no associated Jenking build job
D  - Directory only, not a Maven project
() - Project dependencies
[] - Parent (if not directly under)
```

*Layout:*
```
[*] archon
[X] animal [archon]
	[*] animal-archon [archon] (air, water)
	[X] mammal
		[*] mammal-archon [animal-archon] (sunlight)
		[*] human [mammal-archon] (cow)
		[*] bear [mammal-archon] (salmon)
		[*] cow [mammal-archon] (corn)
		[*] cat [mammal-archon] (sparrow)
	[*] bird [animal-archon]
		[+] sparrow (berry, ant)
		[+] robin (worm)
	[*] fish [animal-archon] (water)
		[+] salmon (fly)
		[+] shark (salmon)
	[X] bug
		[*] insect [animal-archon] (air, water)
		[*] worm [animal-archon] (dirt, water)
		[*] ant [animal-archon] (insect, dirt, grass)
		[*] fly [animal-archon] (insect, cow)
[X] plant [archon]
	[*] plant-archon [archon] (air, water, dirt, sunlight)
	[*] grass [plant-archon]
	[*] berry [plant-archon]
	[*] tree [plant-archon]
	[*] corn [plant-archon]
[D] resource
	[*] air [archon]
	[*] water [archon]
	[*] dirt [archon]
	[*] sunlight [archon]
[*] ecosystem [archon] (depends on all leaf projects above)
```

Use Cases
-------------------------------------------------------------------------------

|Project|Use Case|
|-------|--------|
|archon|Top-level parent project.|
|animal/mammal|Grouping of related projects with a common sub-archon, and various dependencies from the overall project. Each is independently releasable.|
|animal/bird|Releasable parent project with submodules that are not released independently|
|animal/fish|Releasable parent project with submodules that are not released independently. Parent also provides some dependencies.|
|animal/bug|Grouping of related projects where many depend on a sibling project. Each is independently releaseable.|
|plant/*|Related projects with a common archon and no additional dependencies. Each is independently releaseable.|
|resource/*|Basic projects with no common parent and no dependencies.|
|ecosystem|Master build application that pulls in all leaf projects. This is the project that will be cascade-built the most.|

Test Cases
-------------------------------------------------------------------------------

1.	`archon` is updated with new dependencies. All projects need to be updated
	and rebuilt.
	* Release new version of `archon`.
	* Release any sub-archons that depend on the new archon (`animal-archon`,
	  `plant-archon`, `mammal-archon`).
	* Run `mvn versions:update-parent versions:use-latest-snapshots` on entire
	  project to update everything to current snapshots that use the new archon.
	* Cascade-build `ecosystem` project.
2.	A core dependency is updated with new features. All projects that depend
	on it must be rebuilt to take advantage of new features.
	* Update all dependent projects to use current snapshots. For core
	  dependencies (`water`), this may mean updating the entire project
	  tree. For less common dependencies (`insect`) this may mean just
	  updating a subtree (`animal/bug`) and any dependencies (`ecosystem`).
	* Cascade-build `ecosystem` project.
3.	Update a mid-level dependency (i.e. `insect`), but only cascade-release the tree it
	directly affects rather then the entire project.
	* Run `mvn versions:use-latest-snapshots` on animal/bug.
	* Update `ecosystem` manually to use latest snapshots of `animal/bug/fly` and
	  `animal/bug/ant`.
	* Cascade-build `ecosystem` project.

Idea: Efficiently Updating Local Dependencies
-------------------------------------------------------------------------------

How do we update the minimal set of project dependencies for a new release? For
example, when updating `insect`, we only want to update any downstream projects
to avoid excessive releases.

Possibility: new Maven Versions Plugin option to use-latest-snapshots that 
walks the dependency tree of the current project, and updates all dependency
references to the current snapshot for that artifact. It then does the same
for any affected artifacts, converting any of their dependent projects to
use the latest snapshot for them. (Thought: should this be irrespective of
version dependencies, or only affect dependent projects referencing the latest
release?)

Example: if we want to release `insect`, run:
  
`mvn versions:use-latest-snaphots
	-DupdateCascade=true
	-Dincludes=com.barchart.test.jenkinscascade:insect`

This would do the following steps:

1. Update `ant` and `fly` to use the current snapshot version of `insect`
2. Update `sparrow`, `salmon`, and `ecosystem` to use the current snapshot versions of `ant` and `fly`
3. Update `bear`, `cat`, `shark`, and `ecosystem` to use the current snapshot versions for `sparrow` and `salmon`
4. Update `ecosystem` to use the current snapshots for `bear`, `cat`, and`shark`
5. Voila, only the necessary projects have been updated, and `ecosystem` is ready for a cascade build.
