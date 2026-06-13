# **Initial Approach**

When devising my initial framework, I developed three constraints:

1. To yield the largest common circuit between graphs, we want to maximize the anatomical overlap between the selected connectomic datasets. Consequently, I chose FAFB, BANC, and MNCS.  
2. Avoid circuits that may vary between sexes (dimorphism); analyze if the region involved is a circuit related to courtship, mating, or sex-specific behaviors. If so, avoid it.  
3. Look for circuits necessary for survival, likely to have more rigid wiring between individuals.

From these constraints, I decided to focus on finding structures with low individual variance, as there would, in theory, be fewer differences in the synapses and neurons of these structures, allowing for more isomorphic matches between connectomes. These included the central complex, the optic lobe’s columnar lattice, the antennal lobe, and the descending neurons. 

### **Goal Circuit: AVP (OL→ AOTU/MeTu/TuBu → Ring Neurons→ CX)**

## **Mapping FAFB Columnar Lattices**

My vision for this was to map these structures across connectomes, prune down isomorphic violations, and later combine them with an isomorphic path. As the columnar lattice of the right optic lobe has been mostly mapped with FAFB root\_ids, I decided first to attempt mapping both optic lobes in FAFB. For the right optic lobe, I used the neurons from the Visual Columns Challenge to generate the columnar\_lattice subgraph. For the left optic lobe, I created an exclusion mask using the right lobe's nodes (to prevent cross-hemisphere contamination), seeded the remaining network with known columnar cell types, and extracted the largest *weakly connected component (WCC)* to topologically prune disconnected noise and isolate the lattice structure. This worked quite well, and I was able to extract the full lattice (saved in this repository as fafb\_second\_columnar\_lattice.png). 

## **Mapping BANC Columnar Lattices and Matching to FAFB**

Next, to map the lattice in BANC, I built cell signatures, as raw structural mapping across distinct connectomes is an NP-complete problem, and relying solely on unweighted edges and cell typing is insufficient. For the cell signatures, each cell had a two-part profile: 

1. A topological connectivity vector (normalized in-degree and out-degree weights relative to 31 canonical columnar cell classes found in the FAFB columnar lattice).  
2. A neurochemical profile (predicted neurotransmitter probability distributions taken from the BANC 888 Google Bucket).

To isolate the lattice, I calculated the cosine similarity of BANC candidates against the FAFB reference signatures, enforcing a strict typing hierarchy to ground the search: prioritizing established FAFB-typed anchors first, existing BANC types second, and inferred types from label propagation last (KNN logic with synaptic weights).

This was able to extract a partial lattice, with around 16,000 neurons compared to the 23,000 of the FAFB lattice (BANC lattice saved as banc\_columnar\_lattice.png). To achieve isomorphism without triggering a combinatorial explosion (NP-complete), I first employed the Hungarian algorithm (degree as a proxy for local topological identity) to find a 1-to-1 bipartite assignment that minimized local topological cost, followed by a conflict resolution loop. By evaluating the symmetric difference of mapped neighborhoods and pruning the worst structural offenders, the network was forced into a mathematically perfect isomorphic core.

However, I found that by pruning offenders, I no longer had a connected lattice and was left with thousands of islands. While this technically satisfies the challenge’s geometric constraints, it’s not really a “circuit”. I tried several variations of the Hungarian algorithm and my conflict resolution algorithm, as well as using Frank-Wolfe Continuous Relaxation with Fast Approximate QAP and the Gurobi optimization solver with Integer Linear Programming (ILP), but did not find success. 

# **Targeted Seeding, Bridging, and Final Growth**

I then pivoted to a new strategy of finding isomorphic cores, bridging them together, and then using DFS to grow them. This shift from a top-down approach to a bottom-up one proved to be much more effective. I established a rigorously verified, microscopic foundation and algorithmically "grew" the network outward. By constraining the search space to the immediate topological frontier of a perfectly isomorphic seed core, the combinatorics became computationally tractable, allowing the simultaneous alignment of FAFB, BANC, and MCNS connectomes.

## **Seeding**

### **Identifying Isomorphic Cores**

Using a strict three-way symmetric difference check as the driving component of the algorithm, I iterated through candidate triplets across the three subsets of relevant cell types for the target microcircuit (e.g., only CX types or only AOTU/MeTu/TuBu) to verify that incoming and outgoing edges to the existing core are perfectly identical. If a node has a specific connection to the core in one dataset, but lacks that exact corresponding connection in the others, it is pruned. This guarantees the starting core is a mutually isomorphic directed induced subgraph.

### **Growing the Core**

The heuristic here is greedy maximum adjacency, where I scan the frontier of nodes immediately adjacent to the current core and select the single reference candidate that shares the highest number of edges with the already verified core nodes. The idea is that the more connections a candidate has to the established core, the harder it is to find a false positive match in the other datasets, forcing the search down the path of least ambiguity.

Once the reference node is selected, I find its corresponding partners in the other datasets via strict cell-type enforcing. Before any topological math is calculated, I cross-reference the primary biological cell type. This acts as dimensionality reduction, shrinking the candidate pool from hundreds of adjacent neurons to only a handful of biologically viable options. 

To expand the core, I rely on a DFS algorithm. Once a valid triplet is identified that passes both the semantic type filter and the isomorphism check, I temporarily add it to the core and recursively push deeper into the network. To account for extra/missing synapses due to biological noise, this search utilizes a backtracking approach. This ensures I systematically exhaust all valid topological pathways without getting permanently stuck on local maxima.

One important note regarding this expansion is that because the initial seed is already highly targeted, there does not need to be a lot of expansion at this stage. The main goal of the expansion here is to provide the bridging phase with more potential anchor node pairings.

## **Bridging**

To bridge the isolated cores, I shifted from pure topological mapping to a spatially driven heuristic. I utilized BANC coordinates (from banc\_888\_meta.csv) because, with the fewest edges, it is the “limiting connectome” in this situation. The initial pathfinding is conducted by an A\* algorithm constrained by a dynamically generated bounding box. By calculating the minimum and maximum spatial limits around the two cores and enforcing a strict maximum synaptic jump distance, the algorithm restricts the search space to a localized region where a bridge is most likely. Its cost function balances the step count against the remaining physical distance, ensuring the engine explores the most spatially direct routes first rather than wandering aimlessly through the network.

Once a spatial path is charted (exclusively in BANC), the engine enforces topological validation to verify if this exact sequence of connections exists in the other datasets. To account for biological noise, I implemented a soft isomorphism heuristic, where if a newly bridged node possesses a topological conflict with an existing core node, the algorithm calculates the casualties. As long as the number of conflicting core nodes falls below a strict sacrifice threshold, those nodes are pruned away. This tradeoff makes it much easier to discover a bridge and fuse the subgraphs.

Additionally, to prevent the bridge’s isomorphism from collapsing, I check to ensure that newly added nodes in the bridge sequence do not create illegal forward or backward connections with previously established nodes within the bridge path itself. This guarantees that the internal sequence remains purely isomorphic and structurally sound across all three connectomes before the final fusion is accepted.

## **Final Growth**

At this point in the pipeline, I have linked together cores that form my desired end circuit. Now, the goal is to do a local search at each core to expand the circuit and maximize N (neuron count). 

This DFS search engine relies on a multi-tiered filtering pipeline to aggressively prune the search space before the expensive mathematical validation occurs:

### **Topological Lookahead and Terminal Set Checking**

To prevent the engine from blindly mapping nodes that will inevitably fail, I implemented a one-hop topological lookahead, mirroring the terminal set heuristics found in subgraph isomorphism algorithms like VF2. Before a candidate node in the target dataset is considered valid, the engine evaluates its immediate unmapped neighborhood. It cross-references the candidate's connections strictly against the currently expanding frontier. If a candidate does not possess at least an equal number of frontier connections as the reference node, it is impossible for it to support the subsequent expansion of the microcircuit. By identifying this structural deficiency early, the algorithm instantly prunes the branch, bypassing the expensive isomorphism checks.

### **Semantic Classification as Dimensional Reduction**

Building upon the topological lookahead, I integrated a semantic node classification heuristic—a strategy analogous to the VF3 algorithm's use of node attributes to shrink combinatorial search spaces. Rather than relying solely on structural edges, the engine injects biological metadata directly into the candidate generation phase. Before any topological math is calculated, the algorithm enforces cell-type matching. This acts as a massive dimensionality reduction, making the search much less likely to spend time on dead ends.

### **Biological Priority**

To focus my search on the specific structures I’ve set out to discover, the frontier is stratified into two tiers of priorities, managed by a list of cell types (using the fafb\_alignment\_cell\_type from banc\_888\_meta.csv to mitigate differences in set annotations). The engine forces search preference for the prioritized cell types, using degree centrality as a secondary tie-breaker. As an example, if I want to search for the columnar lattice, I set the prioritized types as the canonical types that are found in the lattice itself. 

A critical note here is that it is unadvised to add a secondary priority tier, such as the types of another desired structure(before the degree centrality metric). The neurons with high degree centrality often lead to the discovery of more priority neurons, so without them, making it difficult for the search to return to the priority neuron types.

### **Combinatorial Branch Pruning:**

In highly dense regions, a single reference node might have hundreds of valid isomorphic triplets. Exploring all of them causes the DFS to stall in local degeneracy. To solve this, the engine calculates a deg\_penalty for each valid triplet—scoring them based on how closely their global network degrees match the FAFB reference. It sorts the valid candidates and explicitly truncates the stack pushes to a hard maximum (e.g., 30 branches). This safely cuts away low-probability structural noise and forces the search downward.

### **Iterative State Machine & Checkpoint Sorting:** 

By managing the DFS explicitly on a stack, the algorithm can halt, save its state, and resume. Crucially, upon resuming, the engine executes a score\_state function that re-sorts the stack based on the concentration of high-priority biological cell types and hemispheric alignment, ensuring the search is always focused on that execution’s priority cell types.

# **Final Thoughts & Future Directions**

A limitation I discovered was that BANC 626 is missing edges that FAFB and MCNS have. Since the provided edge list for this challenge was the BANC 626 edge list, I utilized that throughout the project. However, in the future, I would pivot to the BANC 888 and MCNS 1.0 lists (MCNS 0.9 was provided). 

Additionally, with limited time and compute, I was unable to find a bridge on the right hemisphere and connect the right optic lobe to the central complex. This could be due to a combination of the missing edges in BANC 626 or simply the connectivity of the cores I chose in the Seeding phase. Regardless, I am confident that, given more time and resources, my pipeline has the potential to map a circuit to the right optic lobe and perhaps even the other invariant structures previously noted, such as the antennal lobe and descending neurons. 
