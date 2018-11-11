# DSX Local Workshop - User built Python packages + Custom Images on Hadoop

In this workshop, we'll cover:
- Creating python functions in Remote Livy Sessions
- Testing .py files via spark addPyFile
- Packaging a set of .py files into a tar/egg
- Installing custom packages into a DSX Runtime Image within a Project
- Saving and tagging an image as a User
- (Admin) Pushing a user-tagged image (Py Virtual Environment) to HDFS, to be available for all DSX users
- (User) View all available Environments available, and run an example Notebook which leverages an environment with additional Conda + User built packages.


This lab is meant to be instructor-led. That is, the instructor will explain the objectives of the DSX capabilities covered in each lab, and demonstrate some of those capabilities at the beginning of each lab.
## Lab 6. Adding/Importing user py files in Remote Spark Notebooks

### Part 1 - Creating and using py functions in remote sessions.

Let's create a Remote Livy Session, which uses the default python environment available from YARN.
We'll use this to define 2 new helper functions.

**1. Create a new Project "Utils-Workshop"**

**2. Create a new Python 2.7 Notebook "Creating and using py functions in Livy Sessions"**

<img width="370" alt="image" src="img/l6-p1-2.png">

**3. Create a Remote Livy Session**

DSX Includes a set of python helper utils for interacting with remote Hadoop Environments.

First, import `dsx_core_utils` and show a summary of all available DSXHI Endpoints to use within this Notebook.
```
import dsx_core_utils
DSXHI_SYSTEMS = dsx_core_utils.get_dsxhi_info(showSummary=True)
```

Then, define any additional Spark Configurations. See (Livy Sessions REST API)[https://livy.incubator.apache.org/docs/latest/rest-api.html] for additional properties.

Run the following in a cell.
```
myConfig={
 "queue": "default",
 "driverMemory": "2G",
 "numExecutors": 1
}

# Set up sparkmagic to connect to the selected registered HI
# system with the specified configs.
dsx_core_utils.setup_livy_sparkmagic(
  system="<system name shown above>",
  livy="livyspark2",
  addlConfig=myConfig)

# (Re-)load spark magic to apply the new configs.
%reload_ext sparkmagic.magics
```

To see the generated Spark configuration, run `%spark info`.

In a new cell, Start a the remote Livy Session
```
session_name = 'workshop-part1'
livy_endpoint = 'https://value_shown_from_previous_cell/livy2/v1
%spark add -s $session_name -l python -k -u $livy_endpoint
```

<img width="520" alt="image" src="img/l6-p1-3.png">

**4. Creating python functions in Remote Livy Sessions**

Lets create 2 simple functions:

- **run_command** - Simple wrapper to subprocess, to run a linux command within the Driver YARN Container

- **spark_dfs_topandas** - Sample function that takes 2 Spark DFs and returns 2 Pandas DFs. ToPandas() is generally not advisable, as it will be resource intensive on the Spark Driver. Some ML Tools however, require the DataFrame to be a Pandas DF, so this is merelly an example for such scenarios.

```
%%spark -s $session_name
from subprocess import Popen, PIPE, STDOUT

def run_command(command, sleepAfter=None):        
    p = Popen(command, shell=True, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)
    output = p.stdout.read()
    print(output)
    if (sleepAfter != None):
        time.sleep(sleepAfter)

def spark_dfs_topandas(DF1,DF2):
    return DF1.toPandas(),DF2.toPandas()
```

Test run_command, which should return a NodeManager hostname where your Spark Driver is running
```
%%spark -s $session_name

run_command("hostname -f")
run_command("pwd")
```

<img width="520" alt="image" src="img/l6-p1-4.png">

Leave the Livy session open, and proceed to Part 2 in a new tab.
```
%spark cleanup
```

### Part 2. Testing .py files via spark addPyFile

**1. Save the functions as a new Script in the DSX Project**

Navigate to the `Utils-Workshop` project. Select `Scripts` on the Left Hand Side, and "Add Script"

Create a new script named `quicken-demo-utils`

<img width="420" alt="image" src="img/l6-p2-1.png">

With the contents
```
def run_command(command, sleepAfter=None):   
    from subprocess import Popen, PIPE, STDOUT
    p = Popen(command, shell=True, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)
    output = p.stdout.read()
    print(output)
    if (sleepAfter != None):
        time.sleep(sleepAfter)

def spark_dfs_topandas(DF1,DF2):
    return DF1.toPandas(),DF2.toPandas()
```

Click on the Save Icon

<img width="520" alt="image" src="img/l6-p2-1-2.png">

**2. Navigate back to the `Utils-Workshop` Project and Re-open the "Creating and using py functions in Livy Sessions" Notebook.**

**3. Transfer `quicken-demo-utils.py` script which was saved above, to your HDFS user directory.**

- Use cell magic `!ls` to see the relative path of the script which was just saved:

- In a new cell, Show registered WebHDFS Secure URLS which logged in user has access to:
```
import dsx_core_utils
dsx_core_utils.list_dsxhi_webhdfs_endpoints();
```

- Use `dsx_core_utils.hdfs_util.upload_file` to upload a file from DSX to your HDFS desired path
```
dsxlocal_file_location="../scripts/quicken_demo_utils.py"
dsxhi_upload_hdfs_location="/user/user1/quicken_demo_utils.py"
webhdfs_endpoint="https://asgardian-edge.fyre.ibm.com:8443/gateway/jalv-dsx121g-master-1/webhdfs/v1'

<your cluster webhdfs output from above"

dsx_core_utils.hdfs_util.upload_file(webhdfs_endpoint, dsxlocal_file_location, dsxhi_upload_hdfs_location )
```

<img width="420" alt="image" src="img/l6=p2-3.png">


**4. Test the .py file via sc.addPyFile in a new Livy Session¶**

- Delete the old session with %spark cleanup
- Create a new session with %spark add
```
%spark cleanup


%spark add -s $session_name -l python -k -u $livy_endpoint
```

- Test the imported Utils
```
%%spark
sc.addPyFile("hdfs:///user/user1/quicken_demo_utils.py")
```

```
%%spark
import quicken_demo_utils as utils
```

```
%%spark -s $session_name
utils.run_command("hostname -f")
utils.run_command("pwd")
```

<img width="550" alt="image" src="img/l6-p2-4.png">



### Part 3 Create "quicken-lab-utils" python package

Once a set of functions are "stable" and ready to package, you can use a Python setup.py file to create a new "quicken_demo_utils" python package (tar.gz or .egg).

**1. Open a terminal from DSX to a python environment within your project**

Create a new script named `quicken-demo-utils`
<img width="430" alt="image" src="img/l6-p2-1.png">

With the

**2. Create the following Directory structure under your project `misc` directory.**

```
$ pwd
/user-home/1001/DSX_Projects/Quickens-Workshop/misc/workshop-packages

.
└── misc
    └── workshop-packages
        ├── quickens_demo_utils
        │   ├── __init__.py
        │   └── qutils.py
        └── setup.py
```

In which **qutils.py** contains the functions:
```
def run_command(command, sleepAfter=None):        
    from subprocess import Popen, PIPE, STDOUT
    p = Popen(command, shell=True, stdin=PIPE, stdout=PIPE, stderr=STDOUT, close_fds=True)
    output = p.stdout.read()
    print(output)
    if (sleepAfter != None):
        time.sleep(sleepAfter)

def spark_dfs_topandas(DF1,DF2):
    return DF1.toPandas(),DF2.toPandas()
```

**setup.py** contains a simple packaging script:
```
from setuptools import setup, find_packages

setup(
    name = "quickens_demo_utils",
    version = "0.1",
    packages=['quickens_demo_utils']    
)
```    

**__init__.py** is an empty file
```
touch __init__.py
```

**3. Create the dist package**

* sdist will create a tar.gz which can be pip installed or used in Livy

```
python setup.py sdist bdist_egg
```

This will create `dist/quickens_demo_utils-0.1.tar.gz` with a package named `quickens_demo_utils`

<img width="720" alt="image" src="img/part2-n2.png">


**4. Copy the packaged the tar.gz to Packages directory**


```
$ pwd
/user-home/1001/DSX_Projects/Quickens-Workshop/misc/workshop-packages
$ ls dist/
quickens_demo_utils-0.1-py2.7.egg  quickens_demo_utils-0.1.tar.gz
$ cp dist/*.tar.gz /user-home/1001/DSX_Projects/Quickens-Workshop/packages/python/
```


---

## Part 3 - Using  the packged py in Remote Spark Notebooks

### Method 1 HDFS + sc.addFile

**1. Push the .zip to hdfs**

```
# Show registered WebHDFS Secure URLS which logged in user has access to:
import dsx_core_utils
dsx_core_utils.list_dsxhi_webhdfs_endpoints();
```
In a new cell, upload the zip to the remote HDFS endpoint displayed.
```
dsxlocal_file_location="../packages/python/quickens_demo_utils-0.1.tar.gz"
dsxhi_upload_hdfs_location="/user/user1/quickens_demo_utils-0.1.tar.gz"
webhdfs_endpoint="<your cluster webhdfs output from above"

dsx_core_utils.hdfs_util.upload_file(webhdfs_endpoint, dsxlocal_file_location, dsxhi_upload_hdfs_location )
```

**2. Add the zip to an active Livy session**

```
%%spark

sc.addPyFile("/user/user1/quickens_demo_utils-0.1.tar.gz")
```

**3. Run the custom function**

```
%%spark

from quickens_demo_utils import qutils
```

### Method 2 - Installing "quickens_demo_utils" to a custom image in DSX
1. From the **Environments** Tab within your Project, Launch a terminal for the Environment you want to add a package to
<img width="520" alt="image" src="img/part2-m2.png">

**2. Install the custom package we uploaded**

<img width="720" alt="image" src="img/part2-m3.png">

**3. Save the updated image**

<img width="520" alt="image" src="img/part2-m4-1.png">

<img width="320" alt="image" src="img/part2-m5.png">

**4. Test the import in DSX Local Environment to verify**
Once installed into an Environment within a Project, the imports are available within the DSX **Local** Environment.
Cells which run with `%%spark` run remotely. Cells without `%%spark` run locally within DSX.

<img width="750" alt="image" src="img/part2-m6.png">


# Lab 2. Pushing an image with Conda installed + User Created packages.

### Part 1 - Create a custom image with `tqdm` from conda channels

The model creation logic for the following sample requires the python `tqdm` library. Since we will be running the model creation logic in a _remote_ livy session, we will need to create a custom image which includes `tqdm`, and then configure our livy session to use that image.  In order to do this you can take the following steps:

#### A. Start an environment
From your project home page, use the `Environments` tab to _start_ a "`Jupyter with Python 2.7, ...`" environment.

#### B. Install `tqdm` into the environment
From your project home page, use the `Environments` tab to _launch a terminal_ shell for the environment that you started in Step A. When you are inside the terminal, type the following command to install `tqdm`:

```
conda install tqdm -y
```

When the command completes, you can `exit` the terminal.

#### C. Save the environment as a custom image
From your project home page, use the `Environments` tab to _save_ the environment that you edited in Step B.


### Part 2 - Push the image to HDP

#### A. As a user, you can Save your image with *Name* and a *Tag*.

Provide the *Name* and *Tag* to the Hadoop Administrator, as only DSX Administrators have the privileges required to Store this image on Hadoop and make it available for all users.

<img width="550" alt="image" src="img/lab2-pt2-1.png">


#### B. (Admin only) Push the saved image to hadoop
https://content-dsxlocal.mybluemix.net/docs/content/SSAS34_current/local/hadoopintegration.html#hadoopintegration__push

<img width="650" alt="image" src="img/lab2-pt2-2.png">


#### C. (User) Can now Run Weather example using tqdm + workshop-utils imports

Open the `Modeling+Weather+Geographies+on+Hadoop+using+Scikit-Learn.ipynb` notebook in this project to proceed with an example which depends on this new custom image.
