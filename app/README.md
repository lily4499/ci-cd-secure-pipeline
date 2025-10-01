# Login & select your project
#gcloud auth login
gcloud auth login --no-launch-browser
gcloud config set project x-object-472022-q2


# (If you’re brand new) enable required APIs
gcloud services enable container.googleapis.com containerregistry.googleapis.com

# Allow Docker to push to GCR
gcloud auth configure-docker gcr.io

# Create an Autopilot cluster (regional)
gcloud container clusters create-auto demo-autopilot --region us-east4

# Get kubeconfig
gcloud container clusters get-credentials demo-autopilot --region us-east4


# Delete cluster (Autopilot example)
gcloud container clusters delete demo-autopilot --region us-east4 --quiet

