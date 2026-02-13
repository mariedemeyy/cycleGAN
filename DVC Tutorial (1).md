### DVC Tutorial - Local Installation not using Colab

### 1. Requirements Installation

    1.1 Install Git:
        - For Windows: Download Git for Windows and install it.
        - For Linux: Use sudo apt install git.
        - For MacOS: Use brew install git.

    1.2 Install DVC: pip install dvc

    1.3 Verify installations:
        - git --version
        - dvc --version

### 2. Set Up Your Project

    2.1 Create a project directory:
        - mkdir yolo_dvc_project
        - cd yolo_dvc_project

    2.2 Initialize Git and DVC:
        - git init
        - dvc init

    2.3 Commit the initial setup:
        - git add .
        - git commit -m "Initialize Git and DVC"

### 3. Create a Toy YOLO-Format Dataset

    3.1 Set up folders for images and labels:
        - mkdir images
        - mkdir labels

    3.2 Generate mock YOLO data: Run the provided python script called "create_mock_yolo.py"

### 4. Track the Dataset with DVC 

    4.1 Add the dataset to DVC - think about DVC add as this folder is a dataset artifact. Track its content and don’t put it in Git.
        - dvc add images labels
    
    4.2 Commit changes to Git:
        - git add images.dvc labels.dvc .gitignore
        - git commit -m "Track YOLO dataset with DVC"
        - git tag -a "v1.0" -m "model v1.0, 3 images"
        - git status

### 5. Define and Track a Training Pipeline   

    5.1 Create a simple train.py script (use the provided one)

    5.2 dvc stage add writes a pipeline stage named train (specified using the -n option) in dvc.yaml. It tracks all outputs (-o) the same way as dvc add does. Unlike dvc add, dvc stage add also tracks dependencies (-d) and the command (python train.py) that was run to produce the result - This is a reproducible step (pipeline stage): here’s the command, inputs, outputs. If any dependency changed (hash/pointers differs from dvc.lock), DVC re-runs python train.py and regenerates model.pth. If nothing changed, it does nothing.
        - dvc stage add -n train_yolo -d images -d labels -d train.py -o model.pth python train.py

    5.3 Commit the pipeline to Git:
        - git add dvc.yaml
        - git commit -m "Add training pipeline"

### 6. Google Drive Local Storage

When you use DVC, Google Drive becomes a data remote (like a warehouse), not a normal folder.
DVC stores each file by a content hash (looks like random letters/numbers), so the remote contains hashed objects instead of your images/ and labels/ folders. This is expected and it’s how DVC:
 - avoids uploading duplicates across versions,
 - keeps versions consistent,
 - can restore any dataset version later using the .dvc / dvc.lock metadata in Git. 
    
    6.1 Install + verify Google Drive for Desktop
    6.2 Create a remote folder inside your Drive:
        - Create a folder in Google Drive called for example: My Drive/yolo_dvc_project
    6.3 Find the local path to that folder (Windows example): G:\My Drive\yolo_dvc_project
    6.4 Point DVC to that folder:
        - # Add a DVC remote that is just a local folder (inside Google Drive)
            dvc remote add -d gdrive_local "G:\My Drive\DVC_REMOTE_yolo_demo"
    6.5 Commit the remote config to Git:
        - git add .dvc/config
        - git commit -m "Configure DVC remote (Google Drive)"
    6.6 Push current data artifacts to Google Drive:
        - dvc push

### 7. Push to GitHub

    7.1 Push Your Repository to GitHub:
        - Create a new repository on GitHub.

    7.2 Add your remote:
        - git remote add origin https://github.com/<your-username>/<your-repo>.git
        - git branch -M main
        - git push -f origin main

(At this point: GitHub has your repo; Google Drive has your data artifacts)

### 8. Simulate Dataset Changes

Git stores only code + small DVC “pointer” files (dvc.yaml, dvc.lock, *.dvc). The actual images/labels/models are large, so DVC stores them in a data remote (here: a Google Drive folder).
That Drive folder will contain hash-looking files (normal) because DVC stores content by hash to avoid duplicates across versions.

    8.1 Add a new image and label - type python and add the following code:
        
from PIL import Image
img = Image.new("RGB", (256, 256), color="blue")
img.save("images/image_3.jpg")
with open("labels/image_3.txt", "w") as f:
    f.write("1 0.5 0.5 0.4 0.4\n")
    
--> DVC is tracking those folders by content hash (not by timestamps). That means the hash for images/ is different and the hash for labels/ is different

    8.2 Reproduce the pipeline - dvc repro retrains because data changed by asking to rerun dvc stage add
        - dvc repro

    8.3 Save metadata changes in Git (commit the metadata/pointers so Git “remembers” the new dataset/model version):
        - git add dvc.yaml dvc.lock images.dvc labels.dvc .gitignore
        - # optionally include dvc.yaml if you edited pipeline definition:git add dvc.yaml
        - git commit -m "v2: add data and reproduce pipeline"
        - git tag -a "v2.0" -m "model v2.0, 4 images"

    8.4 Push BOTH places (GitHub = pointers + recipe. Google Drive = actual data):
        - git push -u origin main
        - git push --tags
        - dvc push

Note: You can opt not to download and use google drive locally and instead store directly remotely to the website - however you will need to follow this workaround from DVC as google will not let you open their drive using a browser: https://doc.dvc.org/user-guide/data-management/remote-storage/google-drive#using-a-custom-google-cloud-project-recommended

### 9. Switching between workspace versions   

In a team, GitHub is where you share the project recipe (code + DVC metadata like dvc.yaml, dvc.lock, images.dvc, labels.dvc). The actual large data (images/labels/models) is stored in a shared Google Drive folder used as the DVC remote. Each teammate clones the same Git repo, then configures DVC to point to the same shared Drive folder (they must have Drive access), and runs dvc pull to download the exact dataset version referenced by the current Git commit/tag. When someone updates the dataset, they commit/push the updated DVC metadata to Git (git push) and upload the new data objects to Drive (dvc push). Other teammates then do git pull + dvc pull to sync to that version.

    9.1 Switch Git version (metadata):
        - git checkout v1.0

    9.2 Restore the matching data files into your working folder:
        - dvc checkout

    9.3 If files are missing locally, download them from Google Drive:
        - dvc pull

Note: Everytime you change or update your dataset or training pipeline you just repeat steps 8 and 9

### 10. Delete all the cached data. From the repo's root folder:

    - rm -r my_data_folder
    - rm -rf .dvc/cache