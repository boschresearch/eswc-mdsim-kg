## mdsim-Datasets
The *mdsim* datasets are synthetic, anonymized representations of data and domain knowledge found in a MEMS sensor development workflow for a fictional MEMS sensor product.
They emulate the structure and heterogeneity of practical use-case data and include intentionally designed pitfalls that commonly arise when integrating information across sources.
The datasets do not model physical MEMS behavior.

### Available Files
###### CSV
- `Datasets/manufacturing_*.csv` contains in-line manufacturing data for 2 different part types (PX1 & PX2). For each part type there are 1000 wafers simulated, each holding 25 parts
- `Datasets/manufacturing_EOL_*.csv` contains EOL testing data for the parts PX1 & PX2
- `Datasets/engineering.csv` contains in-line data for a new part type variant PX2V2. There are 10 wafers simulated, each holding 25 parts
- `Datasets/engineering_EOL.csv` contains EOL testing data for the part PX2V2
- `Datasets/assembly_map.csv` is a mapping table, tracking what parts are assembled into which sensor. There are 25000 units of sensor SY1 assembled. 
- `Datasets/Final_testing.csv` contains the final test results for the sensor SY1 units
###### Images
- `engineering_coordinates.png` resp. `manufacturing_coordinates.png` visualize wafer coordinate grids used in the simulation for engineering resp. manufacturing
- `DAGs/[name]_dag.svg` visualizes the directed acyclic graph (DAG) used for simulation of the respective part/sensor
- `mdsim_ontology.svg` graph visualization of the *mdsim* ontology modeling the simulated data, a simplified version of the *mdev* ontology
###### RDF
- `mdsim_ontology.ttl` RDF definition of the *mdsim* ontology
- `engineering_mapping_example.ttl` Example RML mapping for the engineering.cvs
- `SHACL/gridalignment.ttl` Example SHACL shapes for flagging wafers & parts with coordinates not matching the ontology standard
- `SHACL/units.ttl` Example SHACL for flagging non qudt units 

## Integration conflicts 
The following conflicts have been injected into the datasets: 
- products are uniquely identified by `usn` NOT by `prod_name`
- parts are uniquely identified by `x_pos`, `y_pos`, `wafer_id`, they have no ID
- `unit:W` is the only valid qudt unit in engineering and manufacturing. All others need preprocessing / conversion
- *PXF_THK_ADJ* uses µm in manufacturing & nm in engineering
- manufacturing adds a new measurement parameter *Bond_ZOffset* to the processing of part *PX2*
- manufacturing/engineering use `prod_name` for part types, assembly uses `sub_prod_name`. (`prod_name` in assembly describes the  sensor type), FT uses `prod_name` for the  type but has no part type info
- engineering uses a different wafer coordinate grid than manufacturing (not center)
- engineering uses Linux timestamp, manufacturing uses datetimestamp
- EOL testing records start & duration, final testing records start & end

## Simulation Overview
The simulation framework implements a **DAG-based causal model** to capture dependencies between process parameters. Each parameter node in the DAG can serve as either an **exogenous input** (controllable parameter) or an **endogenous output** (measurable parameter derived from upstream causes). The causal structure enables systematic propagation of parameter values through the manufacturing process chain. Timestamps and other metadata are tracked and propagated as part of the simulation process as well.

## Parameter Classification
Parameters are categorized into two types:
1. **Controllable parameters**: Design variables sampled from specified bounds using Latin Hypercube Sampling (LHS) to ensure orthogonal coverage of the design space
2. **Measurable parameters**: Endogenous variables computed from upstream causal relationships

## Simulation core steps
The causal computation proceeds as follows:
1. **Initialization**: Sample controllable parameters (LHS)
2. **Topological traversal**: Compute measurable parameters in topological order
3. **Validation**: Check parameter values against specification limits
4. **Postprocessing**: Split and transform into data structures resembling real datasets. Introduce common SICs.

## Causal Effect Propagation
### Topological Ordering
The DAG structure guarantees that all parameters can be computed in a valid topological order, where each node's value depends only on previously computed upstream nodes. This ensures deterministic, reproducible propagation without circular dependencies.
### Edge-Based Causal Models
Each directed edge $(P_i \rightarrow P_j)$ in the causal graph represents a functional relationship between parent parameter $P_i$ and child parameter $P_j$. For the *mdsim*, we use **linear models** as the default.
### Linear Causal Model
For a linear edge relationship, the contribution from parent $P_i$ to child $P_j$ is computed as:
$$c_{i \rightarrow j} = w_{ij} \cdot v_i$$
where:
- $v_i$ is the value of parent parameter $P_i$
- $w_{ij}$ is the edge weight (default: 1.0)

For a child parameter $P_j$ with multiple parents $\{P_1, P_2, ..., P_k\}$, the value is computed as:
$$v_j = b_j + \sum_{i=1}^{k} w_{ij} \cdot v_i + \epsilon_j$$
where:
- $b_j$ is the intercept (bias) term for parameter $j$ (default: 0.0)
- $w_{ij}$ are the edge weights from each parent
- $\epsilon_j \sim \mathcal{N}(0, \sigma_j^2)$ represents measurement noise
