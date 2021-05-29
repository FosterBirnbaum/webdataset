```python
%pylab inline
```

    Populating the interactive namespace from numpy and matplotlib



```python
import zipfile
import tarfile
import os.path
import imageio
import io
import time
import re
import webdataset as wds
```

# Reading the ZipFile

We're taking the data out of the original zip file without unpacking.


```python
src = zipfile.ZipFile("fat.zip")
```


```python
files = [s for s in src.filelist if not s.is_dir()]
```


```python
bydir = {}
for fname in files:
    dir = os.path.dirname(fname.filename)
    bydir.setdefault(dir, []).append(fname)
len(bydir.keys())
```




    330



# Creating the Tar Files

The Falling Things dataset is a collection of videos of falling things. However, there are different ways in which it can be used to create training samples:

- each frame is a training sample, presented in random order
    - applications: pose estimation, stereo, monocular depth
- a small sequence of frames makes up a training sample 
    - applications: optical flow, motion segmentation, frame interpolatino
- each full sequence is a training sample (only 330 training samples)
    - applications: physical modeling, long time tracking
    
For random access datasets, the details of how data is broken up into training samples usually is hidden in the input pipeline. When using large scale, sequential training, this happens in a separate step while we generate training data sets.

Note also that videos in this dataset are not represented as video files but as sequences of frames, distinguished by filename. We are going to do the same thing in the WebDataset representation.

We first generate a full sequence dataset with a special structure of one shard per video:

- one `.tar` file per directory in the ZIP file
- duplicate the camera and object settings for each sample (so that we can later shuffle)
- one sample (=basename) per frame

This is a valid WebDataset, but it also still preserves the video data in sequence within each shard, giving us different processing options.


```python
!rm -rf sequences shuffled
!mkdir sequences shuffled
```


```python
def addfile(dst, name, data):
    assert isinstance(name, str), type(name)
    info = tarfile.TarInfo(name=name)
    info.size = len(data)
    info.uname, info.gname = "bigdata", "bigdata"
    info.mtime = time.time()
    dst.addfile(info, fileobj=io.BytesIO(data))
```


```python
count = 0
for dir in sorted(bydir.keys()):
    tname = f"sequences/falling-things-{count:06d}.tar"
    nfiles = 0
    with tarfile.open(tname, "w|") as dst:
        fnames = bydir[dir]
        print(count, tname, dir)
        camera_settings = [s for s in fnames if "_camera_settings" in s.filename][0]
        camera_settings = src.open(camera_settings, "r").read()
        object_settings = [s for s in fnames if "_object_settings" in s.filename][0]
        object_settings = src.open(object_settings, "r").read()
        last_base = "NONE"
        for finfo in sorted(fnames, key=lambda x: x.filename):
            fname = finfo.filename
            if fname.startswith("_"):
                continue
            base = re.sub(r"\..*$", "", os.path.basename(fname))
            if base != last_base:
                #print("base:", base)
                addfile(dst, dir+"/"+base+".camera.json", camera_settings)
                addfile(dst, dir+"/"+base+".object.json", object_settings)
                last_base = base
            with src.open(fname, "r") as stream:
                data = stream.read()
            addfile(dst, fname, data)
            nfiles += 1
    #break
    count += 1
```

    0 sequences/falling-things-000000.tar fat/mixed/kitchen_0
    1 sequences/falling-things-000001.tar fat/mixed/kitchen_1
    2 sequences/falling-things-000002.tar fat/mixed/kitchen_2
    3 sequences/falling-things-000003.tar fat/mixed/kitchen_3
    4 sequences/falling-things-000004.tar fat/mixed/kitchen_4
    5 sequences/falling-things-000005.tar fat/mixed/kitedemo_0
    6 sequences/falling-things-000006.tar fat/mixed/kitedemo_1
    7 sequences/falling-things-000007.tar fat/mixed/kitedemo_2
    8 sequences/falling-things-000008.tar fat/mixed/kitedemo_3
    9 sequences/falling-things-000009.tar fat/mixed/kitedemo_4
    10 sequences/falling-things-000010.tar fat/mixed/temple_0
    11 sequences/falling-things-000011.tar fat/mixed/temple_1
    12 sequences/falling-things-000012.tar fat/mixed/temple_2
    13 sequences/falling-things-000013.tar fat/mixed/temple_3
    14 sequences/falling-things-000014.tar fat/mixed/temple_4
    15 sequences/falling-things-000015.tar fat/single/002_master_chef_can_16k/kitchen_0
    16 sequences/falling-things-000016.tar fat/single/002_master_chef_can_16k/kitchen_1
    17 sequences/falling-things-000017.tar fat/single/002_master_chef_can_16k/kitchen_2
    18 sequences/falling-things-000018.tar fat/single/002_master_chef_can_16k/kitchen_3
    19 sequences/falling-things-000019.tar fat/single/002_master_chef_can_16k/kitchen_4
    20 sequences/falling-things-000020.tar fat/single/002_master_chef_can_16k/kitedemo_0
    21 sequences/falling-things-000021.tar fat/single/002_master_chef_can_16k/kitedemo_1
    22 sequences/falling-things-000022.tar fat/single/002_master_chef_can_16k/kitedemo_2
    23 sequences/falling-things-000023.tar fat/single/002_master_chef_can_16k/kitedemo_3
    24 sequences/falling-things-000024.tar fat/single/002_master_chef_can_16k/kitedemo_4
    25 sequences/falling-things-000025.tar fat/single/002_master_chef_can_16k/temple_0
    26 sequences/falling-things-000026.tar fat/single/002_master_chef_can_16k/temple_1
    27 sequences/falling-things-000027.tar fat/single/002_master_chef_can_16k/temple_2
    28 sequences/falling-things-000028.tar fat/single/002_master_chef_can_16k/temple_3
    29 sequences/falling-things-000029.tar fat/single/002_master_chef_can_16k/temple_4
    30 sequences/falling-things-000030.tar fat/single/003_cracker_box_16k/kitchen_0
    31 sequences/falling-things-000031.tar fat/single/003_cracker_box_16k/kitchen_1
    32 sequences/falling-things-000032.tar fat/single/003_cracker_box_16k/kitchen_2
    33 sequences/falling-things-000033.tar fat/single/003_cracker_box_16k/kitchen_3
    34 sequences/falling-things-000034.tar fat/single/003_cracker_box_16k/kitchen_4
    35 sequences/falling-things-000035.tar fat/single/003_cracker_box_16k/kitedemo_0
    36 sequences/falling-things-000036.tar fat/single/003_cracker_box_16k/kitedemo_1
    37 sequences/falling-things-000037.tar fat/single/003_cracker_box_16k/kitedemo_2
    38 sequences/falling-things-000038.tar fat/single/003_cracker_box_16k/kitedemo_3
    39 sequences/falling-things-000039.tar fat/single/003_cracker_box_16k/kitedemo_4
    40 sequences/falling-things-000040.tar fat/single/003_cracker_box_16k/temple_0
    41 sequences/falling-things-000041.tar fat/single/003_cracker_box_16k/temple_1
    42 sequences/falling-things-000042.tar fat/single/003_cracker_box_16k/temple_2
    43 sequences/falling-things-000043.tar fat/single/003_cracker_box_16k/temple_3
    44 sequences/falling-things-000044.tar fat/single/003_cracker_box_16k/temple_4
    45 sequences/falling-things-000045.tar fat/single/004_sugar_box_16k/kitchen_0
    46 sequences/falling-things-000046.tar fat/single/004_sugar_box_16k/kitchen_1
    47 sequences/falling-things-000047.tar fat/single/004_sugar_box_16k/kitchen_2
    48 sequences/falling-things-000048.tar fat/single/004_sugar_box_16k/kitchen_3
    49 sequences/falling-things-000049.tar fat/single/004_sugar_box_16k/kitchen_4
    50 sequences/falling-things-000050.tar fat/single/004_sugar_box_16k/kitedemo_0
    51 sequences/falling-things-000051.tar fat/single/004_sugar_box_16k/kitedemo_1
    52 sequences/falling-things-000052.tar fat/single/004_sugar_box_16k/kitedemo_2
    53 sequences/falling-things-000053.tar fat/single/004_sugar_box_16k/kitedemo_3
    54 sequences/falling-things-000054.tar fat/single/004_sugar_box_16k/kitedemo_4
    55 sequences/falling-things-000055.tar fat/single/004_sugar_box_16k/temple_0
    56 sequences/falling-things-000056.tar fat/single/004_sugar_box_16k/temple_1
    57 sequences/falling-things-000057.tar fat/single/004_sugar_box_16k/temple_2
    58 sequences/falling-things-000058.tar fat/single/004_sugar_box_16k/temple_3
    59 sequences/falling-things-000059.tar fat/single/004_sugar_box_16k/temple_4
    60 sequences/falling-things-000060.tar fat/single/005_tomato_soup_can_16k/kitchen_0
    61 sequences/falling-things-000061.tar fat/single/005_tomato_soup_can_16k/kitchen_1
    62 sequences/falling-things-000062.tar fat/single/005_tomato_soup_can_16k/kitchen_2
    63 sequences/falling-things-000063.tar fat/single/005_tomato_soup_can_16k/kitchen_3
    64 sequences/falling-things-000064.tar fat/single/005_tomato_soup_can_16k/kitchen_4
    65 sequences/falling-things-000065.tar fat/single/005_tomato_soup_can_16k/kitedemo_0
    66 sequences/falling-things-000066.tar fat/single/005_tomato_soup_can_16k/kitedemo_1
    67 sequences/falling-things-000067.tar fat/single/005_tomato_soup_can_16k/kitedemo_2
    68 sequences/falling-things-000068.tar fat/single/005_tomato_soup_can_16k/kitedemo_3
    69 sequences/falling-things-000069.tar fat/single/005_tomato_soup_can_16k/kitedemo_4
    70 sequences/falling-things-000070.tar fat/single/005_tomato_soup_can_16k/temple_0
    71 sequences/falling-things-000071.tar fat/single/005_tomato_soup_can_16k/temple_1
    72 sequences/falling-things-000072.tar fat/single/005_tomato_soup_can_16k/temple_2
    73 sequences/falling-things-000073.tar fat/single/005_tomato_soup_can_16k/temple_3
    74 sequences/falling-things-000074.tar fat/single/005_tomato_soup_can_16k/temple_4
    75 sequences/falling-things-000075.tar fat/single/006_mustard_bottle_16k/kitchen_0
    76 sequences/falling-things-000076.tar fat/single/006_mustard_bottle_16k/kitchen_1
    77 sequences/falling-things-000077.tar fat/single/006_mustard_bottle_16k/kitchen_2
    78 sequences/falling-things-000078.tar fat/single/006_mustard_bottle_16k/kitchen_3
    79 sequences/falling-things-000079.tar fat/single/006_mustard_bottle_16k/kitchen_4
    80 sequences/falling-things-000080.tar fat/single/006_mustard_bottle_16k/kitedemo_0
    81 sequences/falling-things-000081.tar fat/single/006_mustard_bottle_16k/kitedemo_1
    82 sequences/falling-things-000082.tar fat/single/006_mustard_bottle_16k/kitedemo_2
    83 sequences/falling-things-000083.tar fat/single/006_mustard_bottle_16k/kitedemo_3
    84 sequences/falling-things-000084.tar fat/single/006_mustard_bottle_16k/kitedemo_4
    85 sequences/falling-things-000085.tar fat/single/006_mustard_bottle_16k/temple_0
    86 sequences/falling-things-000086.tar fat/single/006_mustard_bottle_16k/temple_1
    87 sequences/falling-things-000087.tar fat/single/006_mustard_bottle_16k/temple_2
    88 sequences/falling-things-000088.tar fat/single/006_mustard_bottle_16k/temple_3
    89 sequences/falling-things-000089.tar fat/single/006_mustard_bottle_16k/temple_4
    90 sequences/falling-things-000090.tar fat/single/007_tuna_fish_can_16k/kitchen_0
    91 sequences/falling-things-000091.tar fat/single/007_tuna_fish_can_16k/kitchen_1
    92 sequences/falling-things-000092.tar fat/single/007_tuna_fish_can_16k/kitchen_2
    93 sequences/falling-things-000093.tar fat/single/007_tuna_fish_can_16k/kitchen_3
    94 sequences/falling-things-000094.tar fat/single/007_tuna_fish_can_16k/kitchen_4
    95 sequences/falling-things-000095.tar fat/single/007_tuna_fish_can_16k/kitedemo_0
    96 sequences/falling-things-000096.tar fat/single/007_tuna_fish_can_16k/kitedemo_1
    97 sequences/falling-things-000097.tar fat/single/007_tuna_fish_can_16k/kitedemo_2
    98 sequences/falling-things-000098.tar fat/single/007_tuna_fish_can_16k/kitedemo_3
    99 sequences/falling-things-000099.tar fat/single/007_tuna_fish_can_16k/kitedemo_4
    100 sequences/falling-things-000100.tar fat/single/007_tuna_fish_can_16k/temple_0
    101 sequences/falling-things-000101.tar fat/single/007_tuna_fish_can_16k/temple_1
    102 sequences/falling-things-000102.tar fat/single/007_tuna_fish_can_16k/temple_2
    103 sequences/falling-things-000103.tar fat/single/007_tuna_fish_can_16k/temple_3
    104 sequences/falling-things-000104.tar fat/single/007_tuna_fish_can_16k/temple_4
    105 sequences/falling-things-000105.tar fat/single/008_pudding_box_16k/kitchen_0
    106 sequences/falling-things-000106.tar fat/single/008_pudding_box_16k/kitchen_1
    107 sequences/falling-things-000107.tar fat/single/008_pudding_box_16k/kitchen_2
    108 sequences/falling-things-000108.tar fat/single/008_pudding_box_16k/kitchen_3
    109 sequences/falling-things-000109.tar fat/single/008_pudding_box_16k/kitchen_4
    110 sequences/falling-things-000110.tar fat/single/008_pudding_box_16k/kitedemo_0
    111 sequences/falling-things-000111.tar fat/single/008_pudding_box_16k/kitedemo_1
    112 sequences/falling-things-000112.tar fat/single/008_pudding_box_16k/kitedemo_2
    113 sequences/falling-things-000113.tar fat/single/008_pudding_box_16k/kitedemo_3
    114 sequences/falling-things-000114.tar fat/single/008_pudding_box_16k/kitedemo_4
    115 sequences/falling-things-000115.tar fat/single/008_pudding_box_16k/temple_0
    116 sequences/falling-things-000116.tar fat/single/008_pudding_box_16k/temple_1
    117 sequences/falling-things-000117.tar fat/single/008_pudding_box_16k/temple_2
    118 sequences/falling-things-000118.tar fat/single/008_pudding_box_16k/temple_3
    119 sequences/falling-things-000119.tar fat/single/008_pudding_box_16k/temple_4
    120 sequences/falling-things-000120.tar fat/single/009_gelatin_box_16k/kitchen_0
    121 sequences/falling-things-000121.tar fat/single/009_gelatin_box_16k/kitchen_1
    122 sequences/falling-things-000122.tar fat/single/009_gelatin_box_16k/kitchen_2
    123 sequences/falling-things-000123.tar fat/single/009_gelatin_box_16k/kitchen_3
    124 sequences/falling-things-000124.tar fat/single/009_gelatin_box_16k/kitchen_4
    125 sequences/falling-things-000125.tar fat/single/009_gelatin_box_16k/kitedemo_0
    126 sequences/falling-things-000126.tar fat/single/009_gelatin_box_16k/kitedemo_1
    127 sequences/falling-things-000127.tar fat/single/009_gelatin_box_16k/kitedemo_2
    128 sequences/falling-things-000128.tar fat/single/009_gelatin_box_16k/kitedemo_3
    129 sequences/falling-things-000129.tar fat/single/009_gelatin_box_16k/kitedemo_4
    130 sequences/falling-things-000130.tar fat/single/009_gelatin_box_16k/temple_0
    131 sequences/falling-things-000131.tar fat/single/009_gelatin_box_16k/temple_1
    132 sequences/falling-things-000132.tar fat/single/009_gelatin_box_16k/temple_2
    133 sequences/falling-things-000133.tar fat/single/009_gelatin_box_16k/temple_3
    134 sequences/falling-things-000134.tar fat/single/009_gelatin_box_16k/temple_4
    135 sequences/falling-things-000135.tar fat/single/010_potted_meat_can_16k/kitchen_0
    136 sequences/falling-things-000136.tar fat/single/010_potted_meat_can_16k/kitchen_1
    137 sequences/falling-things-000137.tar fat/single/010_potted_meat_can_16k/kitchen_2
    138 sequences/falling-things-000138.tar fat/single/010_potted_meat_can_16k/kitchen_3
    139 sequences/falling-things-000139.tar fat/single/010_potted_meat_can_16k/kitchen_4
    140 sequences/falling-things-000140.tar fat/single/010_potted_meat_can_16k/kitedemo_0
    141 sequences/falling-things-000141.tar fat/single/010_potted_meat_can_16k/kitedemo_1
    142 sequences/falling-things-000142.tar fat/single/010_potted_meat_can_16k/kitedemo_2
    143 sequences/falling-things-000143.tar fat/single/010_potted_meat_can_16k/kitedemo_3
    144 sequences/falling-things-000144.tar fat/single/010_potted_meat_can_16k/kitedemo_4
    145 sequences/falling-things-000145.tar fat/single/010_potted_meat_can_16k/temple_0
    146 sequences/falling-things-000146.tar fat/single/010_potted_meat_can_16k/temple_1
    147 sequences/falling-things-000147.tar fat/single/010_potted_meat_can_16k/temple_2
    148 sequences/falling-things-000148.tar fat/single/010_potted_meat_can_16k/temple_3
    149 sequences/falling-things-000149.tar fat/single/010_potted_meat_can_16k/temple_4
    150 sequences/falling-things-000150.tar fat/single/011_banana_16k/kitchen_0
    151 sequences/falling-things-000151.tar fat/single/011_banana_16k/kitchen_1
    152 sequences/falling-things-000152.tar fat/single/011_banana_16k/kitchen_2
    153 sequences/falling-things-000153.tar fat/single/011_banana_16k/kitchen_3
    154 sequences/falling-things-000154.tar fat/single/011_banana_16k/kitchen_4
    155 sequences/falling-things-000155.tar fat/single/011_banana_16k/kitedemo_0
    156 sequences/falling-things-000156.tar fat/single/011_banana_16k/kitedemo_1
    157 sequences/falling-things-000157.tar fat/single/011_banana_16k/kitedemo_2
    158 sequences/falling-things-000158.tar fat/single/011_banana_16k/kitedemo_3
    159 sequences/falling-things-000159.tar fat/single/011_banana_16k/kitedemo_4
    160 sequences/falling-things-000160.tar fat/single/011_banana_16k/temple_0
    161 sequences/falling-things-000161.tar fat/single/011_banana_16k/temple_1
    162 sequences/falling-things-000162.tar fat/single/011_banana_16k/temple_2
    163 sequences/falling-things-000163.tar fat/single/011_banana_16k/temple_3
    164 sequences/falling-things-000164.tar fat/single/011_banana_16k/temple_4
    165 sequences/falling-things-000165.tar fat/single/019_pitcher_base_16k/kitchen_0
    166 sequences/falling-things-000166.tar fat/single/019_pitcher_base_16k/kitchen_1
    167 sequences/falling-things-000167.tar fat/single/019_pitcher_base_16k/kitchen_2
    168 sequences/falling-things-000168.tar fat/single/019_pitcher_base_16k/kitchen_3
    169 sequences/falling-things-000169.tar fat/single/019_pitcher_base_16k/kitchen_4
    170 sequences/falling-things-000170.tar fat/single/019_pitcher_base_16k/kitedemo_0
    171 sequences/falling-things-000171.tar fat/single/019_pitcher_base_16k/kitedemo_1
    172 sequences/falling-things-000172.tar fat/single/019_pitcher_base_16k/kitedemo_2
    173 sequences/falling-things-000173.tar fat/single/019_pitcher_base_16k/kitedemo_3
    174 sequences/falling-things-000174.tar fat/single/019_pitcher_base_16k/kitedemo_4
    175 sequences/falling-things-000175.tar fat/single/019_pitcher_base_16k/temple_0
    176 sequences/falling-things-000176.tar fat/single/019_pitcher_base_16k/temple_1
    177 sequences/falling-things-000177.tar fat/single/019_pitcher_base_16k/temple_2
    178 sequences/falling-things-000178.tar fat/single/019_pitcher_base_16k/temple_3
    179 sequences/falling-things-000179.tar fat/single/019_pitcher_base_16k/temple_4
    180 sequences/falling-things-000180.tar fat/single/021_bleach_cleanser_16k/kitchen_0
    181 sequences/falling-things-000181.tar fat/single/021_bleach_cleanser_16k/kitchen_1
    182 sequences/falling-things-000182.tar fat/single/021_bleach_cleanser_16k/kitchen_2
    183 sequences/falling-things-000183.tar fat/single/021_bleach_cleanser_16k/kitchen_3
    184 sequences/falling-things-000184.tar fat/single/021_bleach_cleanser_16k/kitchen_4
    185 sequences/falling-things-000185.tar fat/single/021_bleach_cleanser_16k/kitedemo_0
    186 sequences/falling-things-000186.tar fat/single/021_bleach_cleanser_16k/kitedemo_1
    187 sequences/falling-things-000187.tar fat/single/021_bleach_cleanser_16k/kitedemo_2
    188 sequences/falling-things-000188.tar fat/single/021_bleach_cleanser_16k/kitedemo_3
    189 sequences/falling-things-000189.tar fat/single/021_bleach_cleanser_16k/kitedemo_4
    190 sequences/falling-things-000190.tar fat/single/021_bleach_cleanser_16k/temple_0
    191 sequences/falling-things-000191.tar fat/single/021_bleach_cleanser_16k/temple_1
    192 sequences/falling-things-000192.tar fat/single/021_bleach_cleanser_16k/temple_2
    193 sequences/falling-things-000193.tar fat/single/021_bleach_cleanser_16k/temple_3
    194 sequences/falling-things-000194.tar fat/single/021_bleach_cleanser_16k/temple_4
    195 sequences/falling-things-000195.tar fat/single/024_bowl_16k/kitchen_0
    196 sequences/falling-things-000196.tar fat/single/024_bowl_16k/kitchen_1
    197 sequences/falling-things-000197.tar fat/single/024_bowl_16k/kitchen_2
    198 sequences/falling-things-000198.tar fat/single/024_bowl_16k/kitchen_3
    199 sequences/falling-things-000199.tar fat/single/024_bowl_16k/kitchen_4
    200 sequences/falling-things-000200.tar fat/single/024_bowl_16k/kitedemo_0
    201 sequences/falling-things-000201.tar fat/single/024_bowl_16k/kitedemo_1
    202 sequences/falling-things-000202.tar fat/single/024_bowl_16k/kitedemo_2
    203 sequences/falling-things-000203.tar fat/single/024_bowl_16k/kitedemo_3
    204 sequences/falling-things-000204.tar fat/single/024_bowl_16k/kitedemo_4
    205 sequences/falling-things-000205.tar fat/single/024_bowl_16k/temple_0
    206 sequences/falling-things-000206.tar fat/single/024_bowl_16k/temple_1
    207 sequences/falling-things-000207.tar fat/single/024_bowl_16k/temple_2
    208 sequences/falling-things-000208.tar fat/single/024_bowl_16k/temple_3
    209 sequences/falling-things-000209.tar fat/single/024_bowl_16k/temple_4
    210 sequences/falling-things-000210.tar fat/single/025_mug_16k/kitchen_0
    211 sequences/falling-things-000211.tar fat/single/025_mug_16k/kitchen_1
    212 sequences/falling-things-000212.tar fat/single/025_mug_16k/kitchen_2
    213 sequences/falling-things-000213.tar fat/single/025_mug_16k/kitchen_3
    214 sequences/falling-things-000214.tar fat/single/025_mug_16k/kitchen_4
    215 sequences/falling-things-000215.tar fat/single/025_mug_16k/kitedemo_0
    216 sequences/falling-things-000216.tar fat/single/025_mug_16k/kitedemo_1
    217 sequences/falling-things-000217.tar fat/single/025_mug_16k/kitedemo_2
    218 sequences/falling-things-000218.tar fat/single/025_mug_16k/kitedemo_3
    219 sequences/falling-things-000219.tar fat/single/025_mug_16k/kitedemo_4
    220 sequences/falling-things-000220.tar fat/single/025_mug_16k/temple_0
    221 sequences/falling-things-000221.tar fat/single/025_mug_16k/temple_1
    222 sequences/falling-things-000222.tar fat/single/025_mug_16k/temple_2
    223 sequences/falling-things-000223.tar fat/single/025_mug_16k/temple_3
    224 sequences/falling-things-000224.tar fat/single/025_mug_16k/temple_4
    225 sequences/falling-things-000225.tar fat/single/035_power_drill_16k/kitchen_0
    226 sequences/falling-things-000226.tar fat/single/035_power_drill_16k/kitchen_1
    227 sequences/falling-things-000227.tar fat/single/035_power_drill_16k/kitchen_2
    228 sequences/falling-things-000228.tar fat/single/035_power_drill_16k/kitchen_3
    229 sequences/falling-things-000229.tar fat/single/035_power_drill_16k/kitchen_4
    230 sequences/falling-things-000230.tar fat/single/035_power_drill_16k/kitedemo_0
    231 sequences/falling-things-000231.tar fat/single/035_power_drill_16k/kitedemo_1
    232 sequences/falling-things-000232.tar fat/single/035_power_drill_16k/kitedemo_2
    233 sequences/falling-things-000233.tar fat/single/035_power_drill_16k/kitedemo_3
    234 sequences/falling-things-000234.tar fat/single/035_power_drill_16k/kitedemo_4
    235 sequences/falling-things-000235.tar fat/single/035_power_drill_16k/temple_0
    236 sequences/falling-things-000236.tar fat/single/035_power_drill_16k/temple_1
    237 sequences/falling-things-000237.tar fat/single/035_power_drill_16k/temple_2
    238 sequences/falling-things-000238.tar fat/single/035_power_drill_16k/temple_3
    239 sequences/falling-things-000239.tar fat/single/035_power_drill_16k/temple_4
    240 sequences/falling-things-000240.tar fat/single/036_wood_block_16k/kitchen_0
    241 sequences/falling-things-000241.tar fat/single/036_wood_block_16k/kitchen_1
    242 sequences/falling-things-000242.tar fat/single/036_wood_block_16k/kitchen_2
    243 sequences/falling-things-000243.tar fat/single/036_wood_block_16k/kitchen_3
    244 sequences/falling-things-000244.tar fat/single/036_wood_block_16k/kitchen_4
    245 sequences/falling-things-000245.tar fat/single/036_wood_block_16k/kitedemo_0
    246 sequences/falling-things-000246.tar fat/single/036_wood_block_16k/kitedemo_1
    247 sequences/falling-things-000247.tar fat/single/036_wood_block_16k/kitedemo_2
    248 sequences/falling-things-000248.tar fat/single/036_wood_block_16k/kitedemo_3
    249 sequences/falling-things-000249.tar fat/single/036_wood_block_16k/kitedemo_4
    250 sequences/falling-things-000250.tar fat/single/036_wood_block_16k/temple_0
    251 sequences/falling-things-000251.tar fat/single/036_wood_block_16k/temple_1
    252 sequences/falling-things-000252.tar fat/single/036_wood_block_16k/temple_2
    253 sequences/falling-things-000253.tar fat/single/036_wood_block_16k/temple_3
    254 sequences/falling-things-000254.tar fat/single/036_wood_block_16k/temple_4
    255 sequences/falling-things-000255.tar fat/single/037_scissors_16k/kitchen_0
    256 sequences/falling-things-000256.tar fat/single/037_scissors_16k/kitchen_1
    257 sequences/falling-things-000257.tar fat/single/037_scissors_16k/kitchen_2
    258 sequences/falling-things-000258.tar fat/single/037_scissors_16k/kitchen_3
    259 sequences/falling-things-000259.tar fat/single/037_scissors_16k/kitchen_4
    260 sequences/falling-things-000260.tar fat/single/037_scissors_16k/kitedemo_0
    261 sequences/falling-things-000261.tar fat/single/037_scissors_16k/kitedemo_1
    262 sequences/falling-things-000262.tar fat/single/037_scissors_16k/kitedemo_2
    263 sequences/falling-things-000263.tar fat/single/037_scissors_16k/kitedemo_3
    264 sequences/falling-things-000264.tar fat/single/037_scissors_16k/kitedemo_4
    265 sequences/falling-things-000265.tar fat/single/037_scissors_16k/temple_0
    266 sequences/falling-things-000266.tar fat/single/037_scissors_16k/temple_1
    267 sequences/falling-things-000267.tar fat/single/037_scissors_16k/temple_2
    268 sequences/falling-things-000268.tar fat/single/037_scissors_16k/temple_3
    269 sequences/falling-things-000269.tar fat/single/037_scissors_16k/temple_4
    270 sequences/falling-things-000270.tar fat/single/040_large_marker_16k/kitchen_0
    271 sequences/falling-things-000271.tar fat/single/040_large_marker_16k/kitchen_1
    272 sequences/falling-things-000272.tar fat/single/040_large_marker_16k/kitchen_2
    273 sequences/falling-things-000273.tar fat/single/040_large_marker_16k/kitchen_3
    274 sequences/falling-things-000274.tar fat/single/040_large_marker_16k/kitchen_4
    275 sequences/falling-things-000275.tar fat/single/040_large_marker_16k/kitedemo_0
    276 sequences/falling-things-000276.tar fat/single/040_large_marker_16k/kitedemo_1
    277 sequences/falling-things-000277.tar fat/single/040_large_marker_16k/kitedemo_2
    278 sequences/falling-things-000278.tar fat/single/040_large_marker_16k/kitedemo_3
    279 sequences/falling-things-000279.tar fat/single/040_large_marker_16k/kitedemo_4
    280 sequences/falling-things-000280.tar fat/single/040_large_marker_16k/temple_0
    281 sequences/falling-things-000281.tar fat/single/040_large_marker_16k/temple_1
    282 sequences/falling-things-000282.tar fat/single/040_large_marker_16k/temple_2
    283 sequences/falling-things-000283.tar fat/single/040_large_marker_16k/temple_3
    284 sequences/falling-things-000284.tar fat/single/040_large_marker_16k/temple_4
    285 sequences/falling-things-000285.tar fat/single/051_large_clamp_16k/kitchen_0
    286 sequences/falling-things-000286.tar fat/single/051_large_clamp_16k/kitchen_1
    287 sequences/falling-things-000287.tar fat/single/051_large_clamp_16k/kitchen_2
    288 sequences/falling-things-000288.tar fat/single/051_large_clamp_16k/kitchen_3
    289 sequences/falling-things-000289.tar fat/single/051_large_clamp_16k/kitchen_4
    290 sequences/falling-things-000290.tar fat/single/051_large_clamp_16k/kitedemo_0
    291 sequences/falling-things-000291.tar fat/single/051_large_clamp_16k/kitedemo_1
    292 sequences/falling-things-000292.tar fat/single/051_large_clamp_16k/kitedemo_2
    293 sequences/falling-things-000293.tar fat/single/051_large_clamp_16k/kitedemo_3
    294 sequences/falling-things-000294.tar fat/single/051_large_clamp_16k/kitedemo_4
    295 sequences/falling-things-000295.tar fat/single/051_large_clamp_16k/temple_0
    296 sequences/falling-things-000296.tar fat/single/051_large_clamp_16k/temple_1
    297 sequences/falling-things-000297.tar fat/single/051_large_clamp_16k/temple_2
    298 sequences/falling-things-000298.tar fat/single/051_large_clamp_16k/temple_3
    299 sequences/falling-things-000299.tar fat/single/051_large_clamp_16k/temple_4
    300 sequences/falling-things-000300.tar fat/single/052_extra_large_clamp_16k/kitchen_0
    301 sequences/falling-things-000301.tar fat/single/052_extra_large_clamp_16k/kitchen_1
    302 sequences/falling-things-000302.tar fat/single/052_extra_large_clamp_16k/kitchen_2
    303 sequences/falling-things-000303.tar fat/single/052_extra_large_clamp_16k/kitchen_3
    304 sequences/falling-things-000304.tar fat/single/052_extra_large_clamp_16k/kitchen_4
    305 sequences/falling-things-000305.tar fat/single/052_extra_large_clamp_16k/kitedemo_0
    306 sequences/falling-things-000306.tar fat/single/052_extra_large_clamp_16k/kitedemo_1
    307 sequences/falling-things-000307.tar fat/single/052_extra_large_clamp_16k/kitedemo_2
    308 sequences/falling-things-000308.tar fat/single/052_extra_large_clamp_16k/kitedemo_3
    309 sequences/falling-things-000309.tar fat/single/052_extra_large_clamp_16k/kitedemo_4
    310 sequences/falling-things-000310.tar fat/single/052_extra_large_clamp_16k/temple_0
    311 sequences/falling-things-000311.tar fat/single/052_extra_large_clamp_16k/temple_1
    312 sequences/falling-things-000312.tar fat/single/052_extra_large_clamp_16k/temple_2
    313 sequences/falling-things-000313.tar fat/single/052_extra_large_clamp_16k/temple_3
    314 sequences/falling-things-000314.tar fat/single/052_extra_large_clamp_16k/temple_4
    315 sequences/falling-things-000315.tar fat/single/061_foam_brick_16k/kitchen_0
    316 sequences/falling-things-000316.tar fat/single/061_foam_brick_16k/kitchen_1
    317 sequences/falling-things-000317.tar fat/single/061_foam_brick_16k/kitchen_2
    318 sequences/falling-things-000318.tar fat/single/061_foam_brick_16k/kitchen_3
    319 sequences/falling-things-000319.tar fat/single/061_foam_brick_16k/kitchen_4
    320 sequences/falling-things-000320.tar fat/single/061_foam_brick_16k/kitedemo_0
    321 sequences/falling-things-000321.tar fat/single/061_foam_brick_16k/kitedemo_1
    322 sequences/falling-things-000322.tar fat/single/061_foam_brick_16k/kitedemo_2
    323 sequences/falling-things-000323.tar fat/single/061_foam_brick_16k/kitedemo_3
    324 sequences/falling-things-000324.tar fat/single/061_foam_brick_16k/kitedemo_4
    325 sequences/falling-things-000325.tar fat/single/061_foam_brick_16k/temple_0
    326 sequences/falling-things-000326.tar fat/single/061_foam_brick_16k/temple_1
    327 sequences/falling-things-000327.tar fat/single/061_foam_brick_16k/temple_2
    328 sequences/falling-things-000328.tar fat/single/061_foam_brick_16k/temple_3
    329 sequences/falling-things-000329.tar fat/single/061_foam_brick_16k/temple_4



```python
!ls -lth sequences/*.tar | head
```

    -rw-rw-r-- 1 tmb tmb  46M Aug 30 19:02 sequences/falling-things-000329.tar
    -rw-rw-r-- 1 tmb tmb  46M Aug 30 19:02 sequences/falling-things-000328.tar
    -rw-rw-r-- 1 tmb tmb  26M Aug 30 19:02 sequences/falling-things-000327.tar
    -rw-rw-r-- 1 tmb tmb  32M Aug 30 19:02 sequences/falling-things-000326.tar
    -rw-rw-r-- 1 tmb tmb  33M Aug 30 19:02 sequences/falling-things-000325.tar
    -rw-rw-r-- 1 tmb tmb 155M Aug 30 19:02 sequences/falling-things-000324.tar
    -rw-rw-r-- 1 tmb tmb  89M Aug 30 19:02 sequences/falling-things-000323.tar
    -rw-rw-r-- 1 tmb tmb 165M Aug 30 19:02 sequences/falling-things-000322.tar
    -rw-rw-r-- 1 tmb tmb  90M Aug 30 19:01 sequences/falling-things-000321.tar
    -rw-rw-r-- 1 tmb tmb  99M Aug 30 19:01 sequences/falling-things-000320.tar
    ls: write error: Broken pipe



```python
!tar tf sequences/falling-things-000000.tar | head
```

    fat/mixed/kitchen_0/000000.camera.json
    fat/mixed/kitchen_0/000000.object.json
    fat/mixed/kitchen_0/000000.left.depth.png
    fat/mixed/kitchen_0/000000.left.jpg
    fat/mixed/kitchen_0/000000.left.json
    fat/mixed/kitchen_0/000000.left.seg.png
    fat/mixed/kitchen_0/000000.right.depth.png
    fat/mixed/kitchen_0/000000.right.jpg
    fat/mixed/kitchen_0/000000.right.json
    fat/mixed/kitchen_0/000000.right.seg.png
    tar: write error


# Frame-Level Training

To generate frame level training, we can simply shuffle the sequence data. For this to work, it is important that we associated the sequence level information (camera, object) with each frame, as we did in the construction of the sequence dataset.

We can now shuffle with:

```Bash
$ tarp cat -m 5 -s 500 -o - by-dir/*.tar | tarp split - -o shuffled/temp-%06d.tar
$ tarp cat -m 10 -s 1000 -o - shuffled/temp-*.tar | tarp split - -o shuffled/falling-things-shuffled-%06d.tar
```

If your machine has more memory, you can adjust the `-m` and `-s` options.


```bash
%%bash
tarp cat -m 5 -s 500 -o - sequences/*.tar | tarp split - -o shuffled/temp-%06d.tar && 
tarp cat -m 10 -s 1000 -o - shuffled/temp-*.tar | tarp split - -o shuffled/falling-things-shuffled-%06d.tar &&
rm shuffled/temp-*.tar
```

    [info] # shuffle 500
    [progress] # writing -
    [progress] # source -
    [progress] # shard shuffled/temp-000000.tar
    [progress] # shard shuffled/temp-000001.tar
    [progress] # shard shuffled/temp-000002.tar
    [progress] # shard shuffled/temp-000003.tar
    [progress] # shard shuffled/temp-000004.tar
    [progress] # shard shuffled/temp-000005.tar
    [progress] # shard shuffled/temp-000006.tar
    [progress] # shard shuffled/temp-000007.tar
    [progress] # shard shuffled/temp-000008.tar
    [progress] # shard shuffled/temp-000009.tar
    [progress] # shard shuffled/temp-000010.tar
    [progress] # shard shuffled/temp-000011.tar
    [progress] # shard shuffled/temp-000012.tar
    [progress] # shard shuffled/temp-000013.tar
    [progress] # shard shuffled/temp-000014.tar
    [progress] # shard shuffled/temp-000015.tar
    [progress] # shard shuffled/temp-000016.tar
    [progress] # shard shuffled/temp-000017.tar
    [progress] # shard shuffled/temp-000018.tar
    [progress] # shard shuffled/temp-000019.tar
    [progress] # shard shuffled/temp-000020.tar
    [progress] # shard shuffled/temp-000021.tar
    [progress] # shard shuffled/temp-000022.tar
    [progress] # shard shuffled/temp-000023.tar
    [progress] # shard shuffled/temp-000024.tar
    [progress] # shard shuffled/temp-000025.tar
    [progress] # shard shuffled/temp-000026.tar
    [progress] # shard shuffled/temp-000027.tar
    [progress] # shard shuffled/temp-000028.tar
    [progress] # shard shuffled/temp-000029.tar
    [progress] # shard shuffled/temp-000030.tar
    [progress] # shard shuffled/temp-000031.tar
    [progress] # shard shuffled/temp-000032.tar
    [progress] # shard shuffled/temp-000033.tar
    [progress] # shard shuffled/temp-000034.tar
    [progress] # shard shuffled/temp-000035.tar
    [progress] # shard shuffled/temp-000036.tar
    [progress] # shard shuffled/temp-000037.tar
    [progress] # shard shuffled/temp-000038.tar
    [progress] # shard shuffled/temp-000039.tar
    [progress] # shard shuffled/temp-000040.tar
    [progress] # shard shuffled/temp-000041.tar
    [progress] # shard shuffled/temp-000042.tar
    [progress] # shard shuffled/temp-000043.tar
    [progress] # shard shuffled/temp-000044.tar
    [progress] # shard shuffled/temp-000045.tar
    [progress] # shard shuffled/temp-000046.tar
    [progress] # source -
    [info] # shuffle 1000
    [progress] # writing -
    [progress] # shard shuffled/falling-things-shuffled-000000.tar
    [progress] # shard shuffled/falling-things-shuffled-000001.tar
    [progress] # shard shuffled/falling-things-shuffled-000002.tar
    [progress] # shard shuffled/falling-things-shuffled-000003.tar
    [progress] # shard shuffled/falling-things-shuffled-000004.tar
    [progress] # shard shuffled/falling-things-shuffled-000005.tar
    [progress] # shard shuffled/falling-things-shuffled-000006.tar
    [progress] # shard shuffled/falling-things-shuffled-000007.tar
    [progress] # shard shuffled/falling-things-shuffled-000008.tar
    [progress] # shard shuffled/falling-things-shuffled-000009.tar
    [progress] # shard shuffled/falling-things-shuffled-000010.tar
    [progress] # shard shuffled/falling-things-shuffled-000011.tar
    [progress] # shard shuffled/falling-things-shuffled-000012.tar
    [progress] # shard shuffled/falling-things-shuffled-000013.tar
    [progress] # shard shuffled/falling-things-shuffled-000014.tar
    [progress] # shard shuffled/falling-things-shuffled-000015.tar
    [progress] # shard shuffled/falling-things-shuffled-000016.tar
    [progress] # shard shuffled/falling-things-shuffled-000017.tar
    [progress] # shard shuffled/falling-things-shuffled-000018.tar
    [progress] # shard shuffled/falling-things-shuffled-000019.tar
    [progress] # shard shuffled/falling-things-shuffled-000020.tar
    [progress] # shard shuffled/falling-things-shuffled-000021.tar
    [progress] # shard shuffled/falling-things-shuffled-000022.tar
    [progress] # shard shuffled/falling-things-shuffled-000023.tar
    [progress] # shard shuffled/falling-things-shuffled-000024.tar
    [progress] # shard shuffled/falling-things-shuffled-000025.tar
    [progress] # shard shuffled/falling-things-shuffled-000026.tar
    [progress] # shard shuffled/falling-things-shuffled-000027.tar
    [progress] # shard shuffled/falling-things-shuffled-000028.tar
    [progress] # shard shuffled/falling-things-shuffled-000029.tar
    [progress] # shard shuffled/falling-things-shuffled-000030.tar
    [progress] # shard shuffled/falling-things-shuffled-000031.tar
    [progress] # shard shuffled/falling-things-shuffled-000032.tar
    [progress] # shard shuffled/falling-things-shuffled-000033.tar
    [progress] # shard shuffled/falling-things-shuffled-000034.tar
    [progress] # shard shuffled/falling-things-shuffled-000035.tar
    [progress] # shard shuffled/falling-things-shuffled-000036.tar
    [progress] # shard shuffled/falling-things-shuffled-000037.tar
    [progress] # shard shuffled/falling-things-shuffled-000038.tar
    [progress] # shard shuffled/falling-things-shuffled-000039.tar
    [progress] # shard shuffled/falling-things-shuffled-000040.tar
    [progress] # shard shuffled/falling-things-shuffled-000041.tar
    [progress] # shard shuffled/falling-things-shuffled-000042.tar
    [progress] # shard shuffled/falling-things-shuffled-000043.tar
    [progress] # shard shuffled/falling-things-shuffled-000044.tar
    [progress] # shard shuffled/falling-things-shuffled-000045.tar
    [progress] # shard shuffled/falling-things-shuffled-000046.tar


# Loading the Frame-Level Data


```python
ds = wds.Dataset("shuffled/falling-things-shuffled-000033.tar").decode()
```


```python
for sample in ds:
    break
```


```python
sample.keys()
```




    dict_keys(['__key__', 'object.json', 'right.json', 'left.json', 'left.jpg', 'right.seg.png', 'right.depth.png', 'camera.json', 'right.jpg', 'left.depth.png', 'left.seg.png'])




```python
figsize(12, 8)
subplot(221); imshow(sample["left.jpg"])
subplot(222); imshow(sample["right.jpg"])
subplot(223); imshow(sample["left.seg.png"])
subplot(224); imshow(sample["right.seg.png"])
```




    <matplotlib.image.AxesImage at 0x7f77e56e7f60>




    
![png](falling-things-make-shards_files/falling-things-make-shards_19_1.png)
    



```python
sample["camera.json"]
```




    {'camera_settings': [{'name': 'left',
       'horizontal_fov': 64,
       'intrinsic_settings': {'fx': 768.1605834960938,
        'fy': 768.1605834960938,
        'cx': 480,
        'cy': 270,
        's': 0},
       'captured_image_size': {'width': 960, 'height': 540}},
      {'name': 'right',
       'horizontal_fov': 64,
       'intrinsic_settings': {'fx': 768.1605834960938,
        'fy': 768.1605834960938,
        'cx': 480,
        'cy': 270,
        's': 0},
       'captured_image_size': {'width': 960, 'height': 540}}]}




```python
sample["object.json"]
```




    {'exported_object_classes': ['011_banana_16k'],
     'exported_objects': [{'class': '011_banana_16k',
       'segmentation_class_id': 255,
       'fixed_model_transform': [[-36.39540100097656,
         -17.36479949951172,
         91.50869750976562,
         0],
        [-93.08769989013672, 3.4368999004364014, -36.37120056152344, 0],
        [3.1707000732421875, -98.4207992553711, -17.4153995513916, 0],
        [0.03660000115633011, 1.497499942779541, 0.44449999928474426, 1]],
       'cuboid_dimensions': [19.71739959716797,
        3.8649001121520996,
        7.406599998474121]}]}




```python
sample["left.json"]
```




    {'camera_data': {'location_worldframe': [-487.3075866699219,
       -429.8739929199219,
       208.91009521484375],
      'quaternion_xyzw_worldframe': [0.33379998803138733,
       0.3984000086784363,
       -0.5486999750137329,
       0.6547999978065491]},
     'objects': [{'class': '011_banana_16k',
       'visibility': 0.75,
       'location': [10.001999855041504, 11.863900184631348, 97.93609619140625],
       'quaternion_xyzw': [-0.982200026512146,
        0.12720000743865967,
        -0.04879999905824661,
        -0.1290999948978424],
       'pose_transform_permuted': [[0.06289999932050705,
         -0.2660999894142151,
         -0.961899995803833,
         0],
        [0.9628999829292297, -0.23739999532699585, 0.12870000302791595, 0],
        [0.26249998807907104, 0.9343000054359436, -0.24130000174045563, 0],
        [10.001999855041504, 11.863900184631348, 97.93609619140625, 1]],
       'cuboid_centroid': [10.001999855041504,
        11.863900184631348,
        97.93609619140625],
       'projected_cuboid_centroid': [558.450927734375, 363.05450439453125],
       'bounding_box': {'top_left': [327.92730712890625, 482.7919921875],
        'bottom_right': [396.19769287109375, 636.2205810546875]},
       'cuboid': [[20.235000610351562, 10.343899726867676, 95.17620086669922],
        [1.249899983406067, 15.023799896240234, 92.63909912109375],
        [0.23520000278949738, 11.4128999710083, 93.57170104980469],
        [19.220300674438477, 6.732999801635742, 96.10880279541016],
        [19.768800735473633, 12.314800262451172, 102.30059814453125],
        [0.7835999727249146, 16.994800567626953, 99.76339721679688],
        [-0.23109999299049377, 13.383999824523926, 100.69599914550781],
        [18.754100799560547, 8.704000473022461, 103.23310089111328]],
       'projected_cuboid': [[643.3162231445312, 353.4844970703125],
        [490.36468505859375, 394.5769958496094],
        [481.930908203125, 363.6925048828125],
        [633.6212158203125, 323.8143005371094],
        [628.44189453125, 362.4703063964844],
        [486.0342102050781, 400.8570861816406],
        [478.23748779296875, 372.099609375],
        [619.5501098632812, 334.76629638671875]]}]}




```python
sample["right.json"]
```




    {'camera_data': {'location_worldframe': [-481.3999938964844,
       -428.82489013671875,
       208.91009521484375],
      'quaternion_xyzw_worldframe': [0.33379998803138733,
       0.3984000086784363,
       -0.5486999750137329,
       0.6547999978065491]},
     'objects': [{'class': '011_banana_16k',
       'visibility': 0.75,
       'location': [4.001999855041504, 11.86400032043457, 97.93599700927734],
       'quaternion_xyzw': [-0.982200026512146,
        0.12720000743865967,
        -0.04879999905824661,
        -0.1290999948978424],
       'pose_transform_permuted': [[0.06289999932050705,
         -0.2660999894142151,
         -0.961899995803833,
         0],
        [0.9628999829292297, -0.23739999532699585, 0.12870000302791595, 0],
        [0.26249998807907104, 0.9343000054359436, -0.24130000174045563, 0],
        [4.001999855041504, 11.86400032043457, 97.93599700927734, 1]],
       'cuboid_centroid': [4.001999855041504,
        11.86400032043457,
        97.93599700927734],
       'projected_cuboid_centroid': [511.389404296875, 363.05450439453125],
       'bounding_box': {'top_left': [327.92730712890625, 434.88238525390625],
        'bottom_right': [396.19769287109375, 588.8137817382812]},
       'cuboid': [[14.234999656677246, 10.343899726867676, 95.17620086669922],
        [-4.750100135803223, 15.023900032043457, 92.63899993896484],
        [-5.764800071716309, 11.413000106811523, 93.57160186767578],
        [13.22029972076416, 6.732999801635742, 96.10880279541016],
        [13.768799781799316, 12.314900398254395, 102.30049896240234],
        [-5.216400146484375, 16.99489974975586, 99.76339721679688],
        [-6.231100082397461, 13.383999824523926, 100.6958999633789],
        [12.75409984588623, 8.704000473022461, 103.23310089111328]],
       'projected_cuboid': [[594.8900146484375, 353.4844970703125],
        [440.6123046875, 394.5769958496094],
        [432.6742858886719, 363.6925048828125],
        [585.6649169921875, 323.8143005371094],
        [583.38818359375, 362.4703063964844],
        [439.8346862792969, 400.8570861816406],
        [432.4659118652344, 372.099609375],
        [574.9033813476562, 334.76629638671875]]}]}




```python

```