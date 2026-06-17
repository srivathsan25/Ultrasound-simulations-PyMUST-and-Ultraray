## PyMUST working model
This project implements a two-dimensional ultrasound simulation and beamforming framework utilizing the native PyMUST library. It serves as a legacy, mathematically-driven workflow designed to compute forward acoustic fields and raw signal transformations. Because the execution setup relies heavily on native numerical modeling toolboxes and standardized structural formats, the entire data rendering pipeline is designed to interface seamlessly with target meshes exported from standard workspace directories.

The pipeline initializes by loading a foundational stack of mathematical, visualization, and specialized ultrasound sub-routines. To achieve robust data processing and smooth artifact mitigation, the system sets its computational environment using optimized numpy vectorization and SciPy's isotropic Gaussian filtering routines. Following library declarations, core sensor hardware and medium characteristics are explicitly mapped. Instead of relying on automated preset scripts, the transducer layout is constructed entirely by hand, defining specific aperture configurations, element coordinates, explicit speed of sound metrics, and sampling parameters tailored for spatial imaging.

Once the hardware namespace is configured, the framework leverages a rigorous mesh processing layer to establish the boundaries of the digital scene. It reads structural target definitions—such as identical geometric .obj mesh files—and applies a dynamic scaling routine to convert incoming millimeter coordinates into the meter-based dimensions required by PyMUST. To accommodate a 2D scanning plane, the code employs advanced planar sectioning to reduce the 3D surface vertices into a precise, localized 2D outline through centroid-fallback slicing. This structural reduction directly populates the discrete scatterer positions used to map out the physical boundary profile.

The final phase of the notebook orchestrates the acoustic forward model, complex signal conversion, and matrix beamforming. Using PyMUST's core simus engine, the pipeline translates the localized scatterer coordinates into raw Radio Frequency (RF) echo matrices. These signals are then demodulated into complex baseband In-phase/Quadrature (IQ) formats via the rf2iq module. Finally, the system computes explicit steering properties and passes the baseband matrix through a sparse Delay-and-Sum (dasmtx) execution layer, applying field-of-view angular masks to render finalized, high-contrast B-mode diagnostic images.




## Ultraray working model
This project implements a physics-based ultrasound simulation pipeline utilizing the native UltraRay API.
It is designed as a high-fidelity, high-performance substitute for standard PyMUST simulations by substituting geometric and acoustic approximations with precise optical and acoustic ray-tracing mechanics.
Because the underlying scene management system Resolves spatial geometries and structural configurations via relative pathways, the entire simulation workflow is designed to execute directly out of the root ~/UltraRay/ workspace directory.
The pipeline initializes by configuring path variable overrides and establishing a dedicated computation environment.
To achieve high-efficiency data rendering across multiple channels, the system sets the vectorization and rendering backend using Mitsuba 3 configured to its standalone LLVM CPU variant (llvm_mono), which automatically registers the necessary ultra_ray plugins. Following environment declaration, systemic hardware and media parameters are mapped inside an execution namespace. These mirror a physical linear array probe setup (resembling standard L11-5V configuration metrics), defining a 128-element transducer, specific angular opening apertures, a 50 MHz high-frequency data acquisition sampling rate, 25 discrete plane wave steering angles, and custom medium propagation factors such as an explicit speed of sound ($1540.0 \text{ m/s}$) and attenuation scaling variables.Once parameter blocks are established, the system leverages a core structural layer (SceneLoader) to parse and construct the simulation's physical boundary environment. It ingests target object geometries—such as standardized cylindrical mesh boundaries—and overlays the corresponding structural transducer array file onto the scene layout. This spatial data integration handles the setup for transmit beamforming. Instead of using simplified mathematical grid approximations found in legacy MATLAB toolboxes, the system computes plane wave steering delays directly relative to the synthetic boundaries of the loaded object coordinates, preparing the integrator to perform complex Monte Carlo ray evaluations across more than 100,000 spatial samples.The final segment of the notebook handles system verification and array diagnostics before finalizing data generation. It uses specialized inspection queries to loop through active sensor properties, confirming essential operational criteria like axial resolution, film matrix crop sizes ($128 \times 1$), and maximum scanning distances. It also includes diagnostic handles to track individual elements and normal vector orientations. This ensures the primary acoustic propagation component aligns seamlessly along the depth axis and does not experience spatial matrix skews, paving the way for the successful generation of clean Radio Frequency (RF) and Delay-and-Sum (DAS) beamformed B-mode images.
## PyMUST and UltraRay — Simulation Parameter Comparison
### 1. Transducer Geometry

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Lateral elements | 128 | 128 | Identical |
| Array radius | 30 mm | 30 mm | Identical |
| Angular aperture | 70° | 70° | Identical |
| Element height (elevation) | 4 mm | 4 mm | Identical |
| Elevation discretization | Not modeled | 1 element | Methodological difference — PyMUST simulates a single 2-D imaging plane; UltraRay traces rays through the full 3-D scene |

### 2. Acquisition Parameters

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Center frequency | 5 MHz | 5 MHz | Identical |
| Sampling frequency | 50 MHz | 50 MHz | Identical |
| Transmit pulse length | 3 cycles | 3 cycles | Identical |
| Speed of sound | 1540 m/s | 1540 m/s | Identical |
| Attenuation coefficient | 0.1 | 0.1 | Identical value; underlying attenuation model differs (wave-equation damping term vs. ray/path energy loss term) |
| Transducer fractional bandwidth | 70% | No corresponding parameter | Not applicable — required by PyMUST's pulse-echo response model; UltraRay exposes no equivalent setting |

### 3. Transmit Sequence

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Steered transmissions | 25 | 25 | Identical |
| Steering angle range | -30° to +30° | -30° to +30° | Identical |
| Transmit apodization | Hann taper across the aperture | Not present in the configuration reviewed | Unverified — apodization is not exposed in UltraRay's argument list; its absence here does not confirm its absence in the renderer |
| Inter-angle compounding weighting | Hann-weighted | Implementation internal to `beamform()`, not inspected | Unverified |

### 4. Forward Simulation Model

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Simulation method | Deterministic linear-acoustics point-scatterer model | Monte Carlo ray/path tracing | Methodological difference (fundamental to the comparison) |
| Object representation | Discrete scatterer cloud sampled from the mesh cross-section | Continuous triangle mesh, intersected directly | Methodological difference |
| Samples per render | Not applicable | 100,000 | No PyMUST equivalent |
| Maximum ray bounce depth | Not applicable | 10 | No PyMUST equivalent |
| Maximum ray travel distance | Not applicable | 0.20 m | No PyMUST equivalent |
| Temporal binning filter | Not applicable | Box filter, σ = 2.0 samples | No PyMUST equivalent |

### 5. Image Reconstruction

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Beamforming algorithm | Delay-and-sum | Delay-and-sum | Identical principle, independent implementations |
| Receive f-number | 2.5 | 28 | Intentional mismatch — UltraRay requires a narrow receive aperture to suppress Monte Carlo sampling noise; PyMUST requires a comparatively wide aperture to control sidelobe energy from an element pitch coarser than λ/2. The two values address unrelated artifact sources |
| Reconstruction grid | 420 × 320 px, fixed extent | Approximately 3426 × 4845 px, internally determined | Not matched |

### 6. Display

| Parameter | PyMUST | UltraRay | Status |
|---|---|---|---|
| Dynamic range | 60 dB | 60 dB | Identical |
| Colormap | Grayscale | Grayscale | Identical |
| Depth axis orientation | Increasing downward | Increasing downward | Identical |




