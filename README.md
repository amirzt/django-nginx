# nginx
this is a simple step by step guide to deploy a django restframework api on an ubunto vps using nginx and gunicorn without to much explanation and just the steps<br />
back-end : django rest framework<br />
database : mysql<br />
vps : ubunto<br />
using nginx and gunicorn<br />

# set up ubunto
##
    sudo apt update
##
    sudo apt upgrade
##
    sudo apt install python3-venv python3-dev libpq-dev nginx curl
make working directory <br />

##
    mkdir api
