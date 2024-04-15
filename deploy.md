# Deploying to an ubuntu server 
Though out this process i will be using a linux server with the username of `bee`


### Upgrade system
run `sudo apt update` and `sudo apt update` 

### install required packages

```
sudo apt install python3-venv
```

```
sudo apt install libpq-dev
```

```
sudo apt install nginx
```

```
sudo apt install nginx 
```

```bash
sudo apt install curl
```

### Create a new virtual environment 
ensure you are in the root directory of your linux server i.e `/home/<username>/`

```bash
python3 -m venv venv
```

### Activate your virtual environment
```bash
source venv/bin/activate
```
(venv) should now be prepended to your path, this means the virtual environment was activated successfully

### Clone the github repo
```
git clone https://github.com/Codebee50/NNRA-circle.git 
```

### Install requirements
Navigate into the folder containing requirements.txt file
```bash
cd NNRA-circle
```

```
pip install -r requirements.txt
```












