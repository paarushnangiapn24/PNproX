PNproX
A fast secure, encrypted web browser
git init

git add .

git commit -m "Initial PN24pro publish"

gh repo create OWNER/REPO --public --source=. --push

chmod +x publish_full.sh

./publish_full.sh OWNER REPO
