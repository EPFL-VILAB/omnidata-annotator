# Snowball Annotator


## Installation
We provide a docker that contains the code and all the necessary libraries. It's simple to install and run.

```bash
docker pull snowball99/snowball:latest
docker run -ti --rm  snowball99/snowball:latest
```
The code is now available in the docker under your home directory (`/usr/Snowball`), and all the necessary libraries should already be installed in the docker. We also include a sample mesh from Replica dataset in `/usr/Snowball/frl_apartment_0/mesh.ply` which can be used as the input to the annotator for running the Demo.


## Run Demo
| Surface Normals | Euclidean Depth | Semantics  |
| :-------------: |:-------------:| :-----:|
| ![](./assets/replica/point_0009_view_equirectangular_domain_normal.png) | ![](./assets/replica/point_0009_view_equirectangular_domain_depth_euclidean.png) | ![](./assets/replica/point_0009_view_equirectangular_domain_semantic.png) |
| ![](./assets/replica/point_0010_view_equirectangular_domain_normal.png) | ![](./assets/replica/point_0010_view_equirectangular_domain_depth_euclidean.png) | ![](./assets/replica/point_0010_view_equirectangular_domain_semantic.png)


Next, we show how to generate the mid-level cues from a sample mesh:

```bash
RGB (8-bit)              Surface Normals (8-bit)     Principal Curvature (8-bit)
Re(shading) (8-bit)      Depth Z-Buffer (16-bit)     Depth Euclidean (16-bit)
Texture Edges (16-bit)   Occlusion Edges (16-bit)    Keypoints 2D (16-bit)
Keypoints 3D (16-bit)    2D Segmentation (8-bit)     2.5D Segmentation (8-bit)
Semantic Segmentation (8-bit)
```

**To generate a specific mid-level cue with the annotator, use a command in the format below:**
``` bash

./omnidata-annotator.sh --model_path=$PATH_TO_FOLDER_OF_MODEL --task=$TASK with {SETTING=VALUE}*

```

The `--model_path` tag specifies the path to the folder containing the mesh, where the data from all other mid-level cues will be saved, and the `--task` tag specifies the target mid-level cue.
You can specify different setting values for each task in the command. The list of all settings defined for different mid-level cues is found in `settings.py`.

The final folder structure will be as follows:
```bash
model_path
│   mesh.ply
│   mesh_semantic.ply
│   texture.png
│   camera_poses.json   
└─── point_info
└─── rgb
└─── normals
└─── depth_zbuffer
│   ...
│   
└─── pano
│   └─── point_info
│   └─── rgb
│   └─── normal
│   └─── depth_euclidean
```

Now, we run the annotator for different tasks.

### 1. Camera and Point-of-Interest Sampling:
Camera poses can be provided by a `json` file (if the mesh comes with aligned RGB), or you can generate dense camera locations using the pipeline with `Poisson Disc Sampling`. Points-of-interest are then sampled from the mesh subject to multi-view constraints.

#### Read camera poses from json file:
The following command samples 100 points-of-interest using the camera poses defined in `camera_poses.json`.
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=points \
   with GENERATE_CAMERAS=False CAMERA_POSE_FILE=camera_poses.json \
        MIN_VIEWS_PER_POINT=3  NUM_POINTS=100 \
        MODEL_FILE=mesh.ply    POINT_TYPE=CORRESPONDENCES
```
In order to read the camera poses from the json file, you should specify `GENERATE_CAMERAS=False`. This json file should contain `location` and `quaternion rotation (wxyz)` for a list of cameras. Below, you can see how this information should be saved for each camera.

```javascript
{
    "camera_id": "0000",
    "location": [0.2146, 1.2829, 0.2003],
    "rotation_quaternion": [0.0144, -0.0100, -0.0001,-0.9998]
}
```
You can specify the type of generated points. `POINT_TYPE=CORRESPONDENCES` is used for generating fixated views of the points. Switch to `POINT_TYPE=SWEEP` in case you want to generate panoramas.
#### Sample dense camera poses:
You can sample new camera poses before sampling the points-of-interest using `GENERATE_CAMERAS=True`.
There are 2 ways of generating the camera poses depending on wether the mesh is a scene (like in Replica) or is an object (Google Scanned Objects).

##### Sample camera poses inside a scene:
In this case, you have to specify `SCENE=True`.

```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=points \
   with GENERATE_CAMERAS=True     SCENE=True \
        MIN_CAMERA_HEIGHT=1       MAX_CAMERA_ROLL=10 \
        MIN_CAMERA_DISTANCE=1     MIN_CAMERA_DISTANCE_TO_MESH=0.3 \
        MIN_VIEWS_PER_POINT=3     POINTS_PER_CAMERA=5 \
        MODEL_FILE=mesh.ply       POINT_TYPE=CORRESPONDENCES

```
Camera locations are sampled inside the mesh using **Poisson Disc Sampling** to cover the space. Minimum distance between cameras is specified by `MIN_CAMERA_DISTANCE`. `MIN_CAMERA_DISTANCE_TO_MESH` defines the minimum distance of each camera to the closest point of the mesh.
Camera `yaw` is sampled uniformly in `[-180°, 180°]`, camera `roll` comes from a truncated normal distribution in `[-MAX_CAMERA_ROLL, MAX_CAMERA_ROLL]`, and camera `pitch` will be specified automatically when fixating the camera on a point-of-interest.
More camera settings such as  `MIN_CAMERA_HEIGHT`,  `MAX_CAMERA_HEIGHT`, etc. are defined in `settings.py`.
You can specify the number of generated points either by `NUM_POINTS` or `NUM_POINTS_PER_CAMERA`. In case we have `NUM_POINTS=None`, the number of generated points will be `NUM_POINTS_PER_CAMERA * number of cameras`.

##### Generate camera poses for an object:
If the mesh is an object you have to specify `SCENE=False`. In this case, camera locations will be sampled on a `sphere` surrounding the mesh. `SPHERE_SCALING_FACTOR` will specify the scaling factor of this sphere relative to the smallest bounding sphere of the mesh. You can specify the number of generated cameras by `NUM_CAMERAS`. Camera rotations will be sampled the same as above.

```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/google_scanned_objects/TEA_SET --task=points \
  with GENERATE_CAMERAS=True    SCENE=False \
       NUM_CAMERAS=12           MAX_CAMERA_ROLL=10 \
       POINTS_PER_CAMERA=5      MIN_VIEWS_PER_POINT=3 \
       MODEL_FILE=model.obj     SPHERE_SCALING_FACTOR=2
```

|  |  |  |  |  |
| :-------------: |:-------------:|:-------------:|:-------------:|:-------------:|
| ![](./assets/google-objects/point_11_view_0_domain_rgb.png) | ![](./assets/google-objects/point_19_view_4_domain_rgb.png) | ![](./assets/google-objects/point_21_view_0_domain_rgb.png) |![](./assets/google-objects/point_22_view_0_domain_rgb.png) | ![](./assets/google-objects/point_29_view_0_domain_rgb.png) 

### 2. Surface Normals:

In order to generate surface normal images simply run:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=normal \
    with MODEL_FILE=mesh.ply  CREATE_FIXATED=True
```
This will generate fixated views.

| Replica | Clevr | Google Objects  | Replica+GSO  | BlendedMVG  |
| :-------------: |:-------------:|:-------------:|:-------------:|:-------------:|
| ![](./assets/replica/point_246_view_34_domain_rgb.png) | ![](./assets/clevr/point_55_view_0_domain_rgb.png) | ![](./assets/google-objects/point_28_view_1_domain_rgb.png) |![](./assets/replica-gso/point_754_view_16_domain_rgb.png) | ![](./assets/blendedMVG/00000253.jpg) |
| ![](./assets/replica/point_246_view_34_domain_normal.png) | ![](./assets/clevr/point_55_view_0_domain_normal.png) | ![](./assets/google-objects/point_28_view_1_domain_normal.png)|![](./assets/replica-gso/point_754_view_16_domain_normal.png) | ![](./assets/blendedMVG/point_253_view_0_domain_normal.png)


In case you want to generate panoramas switch to `CREATE_FIXATED=False`  and `CREATE_PANOS=True`:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=normal \
    with MODEL_FILE=mesh.ply CREATE_FIXATED=False CREATE_PANOS=True
```
![](/home/ainaz/Desktop/EPFL/replica_apartment_0/pano/normal/point_0019_view_equirectangular_domain_normal.png)

### 3. Depth ZBuffer:
To generate depth zbuffer images :
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=depth_zbuffer \
    with MODEL_FILE=mesh.ply  DEPTH_ZBUFFER_MAX_DISTANCE_METERS=16
```
ZBuffer depth is defined as the distance to the camera plane. The depth sensitivity is specified by the maximum depth in meters. With 16-bit images and `DEPTH_ZBUFFER_MAX_DISTANCE_METERS` equal to 16m, the depth sensitivity will be 16 / 2^16 = 1/4096 meters. Pixels with maximum depth value (2^16) indicate the invalid parts of the image (such as mesh holes).

| Replica | Google Objects  | Hypersim  | BlendedMVG  |
| :-------------:|:-------------:|:-------------:|:-------------:|
| ![](./assets/replica/point_156_view_10_domain_rgb.png) | ![](./assets/google-objects/point_21_view_5_domain_rgb.png) |![](./assets/hypersim/point_85_view_0_domain_rgb.png) | ![](./assets/blendedMVG/point_1006_view_0_domain_rgb.png) |
| ![](./assets/replica/point_156_view_10_domain_depth_zbuffer.png) | ![](./assets/google-objects/point_21_view_5_domain_depth_zbuffer.png)|![](./assets/hypersim/point_85_view_0_domain_depth_zbuffer2.png) | ![](./assets/blendedMVG/point_1006_view_0_domain_depth_zbuffer.png)


### 4. Depth Euclidean:
To generate depth euclidean images :
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=depth_euclidean \
    with MODEL_FILE=mesh.ply  DEPTH_EUCLIDEAN_MAX_DISTANCE_METERS=16
```
Euclidean depth is measured as the distance from each pixel to the camera’s optical center. You can specify depth sensitivity the same as depth Zbuffer.

| Taskonomy | Clevr  | BlendedMVG  |
| :-------------:|:-------------:|:-------------:|
| ![](./assets/taskonomy/point_21_view_2_domain_rgb.png) | ![](./assets/clevr/point_2368_view_0_domain_rgb.png) |![](./assets/blendedMVG/point_979_view_0_domain_rgb.png) |
| ![](./assets/taskonomy/point_21_view_2_domain_depth_euclidean2.png) | ![](./assets/clevr/point_2368_view_0_domain_depth_euclidean2.png) | ![](./assets/blendedMVG/point_979_view_0_domain_depth_euclidean.png)

### 5. Re(shading):
To generate reshading images :
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=reshading \
    with MODEL_FILE=mesh.ply  LAMP_ENERGY=2.5
```


| Taskonomy | Google Objects  | Hypersim  |
| :-------------:|:-------------:|:-------------:|
| ![](./assets/taskonomy/point_202_view_5_domain_rgb.png) | ![](./assets/google-objects/point_5_view_2_domain_rgb_new.png) | ![](./assets/hypersim/point_85_view_0_domain_rgb.png) |
| ![](./assets/taskonomy/point_202_view_5_domain_reshading.png) | ![](./assets/google-objects/point_5_view_2_domain_reshading.png) | ![](./assets/hypersim/point_85_view_0_domain_reshading.png)

### 6. Principal Curvature:
To generate principal curvature run:

```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=curvature with MIN_CURVATURE_RADIUS=0.03
```

| Taskonomy | Replica |
| :---:|:---:|
| ![](./assets/taskonomy/point_202_view_5_domain_rgb.png) | ![](./assets/replica/point_0_view_2_domain_rgb.png) |
| ![](./assets/taskonomy/point_202_view_5_domain_principal_curvature.png) | ![](./assets/replica/point_0_view_2_domain_principal_curvature.png) 

### 7. Keypoints 2D:
2D keypoints are generated from corresponding `RGB` images for each point and view. You can generate 2D keypoint images using the command below :

```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=keypoints2d
```

### 8. Keypoints 3D:
3D keypoints are similar to 2D keypoints except that they are derived from 3D data. Therefore you have to generate `depth_zbuffer` images before generating 3D keypoints.
To generate 3D keypoint images use the command below:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=keypoints3d \
    with KEYPOINT_SUPPORT_SIZE=0.3
```
`KEYPOINT_SUPPORT_SIZE` specifies the diameter of the sphere around each 3D point that is used to decide if the point should be a keypoint. 0.3 meters is suggested for indoor spaces.

| Replica | Clevr | Hypersim  | BlendedMVG  |
| :-------------: |:-------------:|:-------------:|:-------------:|
| ![](./assets/replica/point_47_view_25_domain_rgb.png) | ![](./assets/clevr/point_2368_view_0_domain_rgb.png) |![](./assets/hypersim/point_85_view_0_domain_rgb.png) | ![](./assets/blendedMVG/point_4_view_0_domain_rgb.png) |
| ![](./assets/replica/point_47_view_25_domain_keypoints3d.png) | ![](./assets/clevr/point_2368_view_0_domain_keypoints3d.png) | ![](./assets/hypersim/point_85_view_0_domain_keypoints3d.png) | ![](./assets/blendedMVG/point_4_view_0_domain_keypoints3d.png)


### 9. Texture Edges:
Texture(2D) Edges are computed from corresponding `RGB` images using **Canny edge detection** algorithm. To generate 2D edges:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=edge2d \
    with CANNY_RGB_BLUR_SIGMA=3.0
```
`CANNY_RGB_BLUR_SIGMA` specifies the sigma in Gaussian filter used in Canny edge detector.

| Replica | Clevr | Replica+GSO |
| :-------------: |:-------------:|:-------------:|
| ![](./assets/replica/point_47_view_25_domain_rgb.png) | ![](./assets/clevr/point_2368_view_0_domain_rgb.png) |![](./assets/replica-gso/point_74_view_19_domain_rgb.png)|
| ![](./assets/replica/point_47_view_25_domain_edge_texture2.png) | ![](./assets/clevr/point_2368_view_0_domain_edge_texture2.png) | ![](./assets/replica-gso/point_74_view_19_domain_edge_texture2.png) 

### 10. Occlusion Edges:
Occlusion(3D) Edges are derived from `depth_zbuffer` images, so you have to generate those first. To generate 3D edges :
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=edge3d
  with EDGE_3D_THRESH=None
```

### 11. 2D Segmentation:
2D Segmentation images are generated using **Normalized Cut** algorithm from corresponding `RGB` images:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=segment2d \
  with SEGMENTATION_2D_BLUR=3     SEGMENTATION_2D_CUT_THRESH=0.005  \
       SEGMENTATION_2D_SCALE=200  SEGMENTATION_2D_SELF_EDGE_WEIGHT=2
```


### 11. 2.5D Segmentation:
2.5D Segmentation uses the same algorithm as 2D, but the labels are computed jointly from `occlusion edges`, `depth zbuffer `, and `surface normals`. 2.5D segmentation incorporates information about the scene geometry that is not directly present in the `RGB` image.
To generate 2.5D segmentation images :
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=segment25d \
  with SEGMENTATION_2D_SCALE=200        SEGMENTATION_25D_CUT_THRESH=1  \
       SEGMENTATION_25D_DEPTH_WEIGHT=2  SEGMENTATION_25D_NORMAL_WEIGHT=1 \
       SEGMENTATION_25D_EDGE_WEIGHT=10
```
You can specify the weights for each of the `occlusion edges`, `depth zbuffer`, and `surface normal` images used in 2.5D segmentation algorithm by `SEGMENTATION_25D_EDGE_WEIGHT`, `SEGMENTATION_25D_DEPTH_WEIGHT`, and `SEGMENTATION_25D_NORMAL_WEIGHT` respectively.


| Replica | Google Objects | Hypersim |
| :-------------: |:-------------:|:-------------:|
|<img width=50/>|<img width=50/>|<img width=50/>|
| ![](./assets/replica/point_300_view_0_domain_rgb.png) | ![](./assets/google-objects/point_21_view_5_domain_rgb.png) |![](./assets/hypersim/point_85_view_0_domain_rgb.png)|
| ![](./assets/replica/point_300_view_0_domain_segment_unsup25d.png) | ![](./assets/google-objects/point_21_view_5_domain_segment_unsup25d.png) | ![](./assets/hypersim/point_85_view_0_domain_segment_unsup25d.png) 

### 12. Semantic Segmentation:
In order to generate semantic photos, we need to have mesh face colors. These colors should be saved as a face property named `color` in the `.ply` file.

Below, you can see how this `color` property should be saved in `mesh_semantic.ply`.

```bash
>>> from plyfile import PlyData
>>> model = PlyData.read('mesh_semantic.ply')
>>> faces = model.elements[1]
>>> faces.properties
(PlyListProperty('vertex_indices', 'uchar', 'int'), PlyListProperty('color', 'uchar', 'int'))
>>> faces['color']
array([array([ 96, 118,  19], dtype=int32),
       array([ 48, 110, 165], dtype=int32),
       array([ 96, 118,  19], dtype=int32), ...,
       array([10, 10, 10], dtype=int32), array([10, 10, 10], dtype=int32),
       array([10, 10, 10], dtype=int32)], dtype=object)
>>>
```
To generate semantic segmentation photos, run the command below:
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/frl_apartment_0 --task=semantic \
  with SEMANTIC_MODEL_FILE=mesh_semantic.ply
```
Notice that you should specify the value for `SEMANTIC_MODEL_FILE` instead of `MODEL_FILE` which was used for other tasks.

| Replica | Taskonomy  | Replica+GSO  | Hypersim  |
| :-------------:|:-------------:|:-------------:|:-------------:|
| ![](./assets/replica/point_246_view_34_domain_rgb.png) | ![](./assets/taskonomy/point_202_view_5_domain_rgb.png) |![](./assets/replica-gso/point_74_view_19_domain_rgb.png) | ![](./assets/hypersim/point_0_view_0_domain_rgb.png) |
| ![](./assets/replica/point_246_view_34_domain_semantic3.png) | ![](./assets/taskonomy/point_202_view_5_domain_semantic3.png)|![](./assets/replica-gso/point_74_view_19_domain_semantic3.png) | ![](./assets/hypersim/point_0_view_0_domain_semantic.png)


### 13. RGB:
There are several ways to generate rgb images.
#### Generate rgb images using texture UV map:
You can generate `rgb` images for each point and view using texture UV map. You should specify the texture file using `TEXTURE_FILE`.
```bash
./omnidata-annotate.sh --model_path=/usr/omnidata-annotator/google_scanned_objects/Vtech_Stack_Sing_Rings_636_Months --task=rgb \
    with MODEL_FILE=mesh.ply  CREATE_FIXATED=True \
         USE_TEXTURE=True     TEXTURE_FILE=texture.png
```
|  |  |  |  | 
| :-------------: |:-------------:|:-------------:|:-------------:|
| ![](./assets/google-objects/point_5_view_2_domain_rgb_new.png) | ![](./assets/google-objects/point_22_view_0_domain_rgb.png) | ![](./assets/google-objects/point_21_view_5_domain_rgb.png) |![](./assets/google-objects/point_28_view_1_domain_rgb.png)
