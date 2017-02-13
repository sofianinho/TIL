## Random git commands...
... that could have been a simple `gist` entry.

### Show/log to know the stats about the files that changed

1. Know which git commit number you want to detail
  ```sh
  git log
  # Other very useful subcommands of log are explained in the manual, for example
  # I want to know the git commits that affected files in a folder between two identified commit IDs
  git log 4c3eedb5fdddea9bcde99c327ac77d202d3d7d3d..7206de613e403f6addc7f56a8974c3b1d5f606fd web/PoCRANaaS/
  ```

2. Know which files were changed in that commit 
  ```sh
  # To have the big picture of changes in a folder
  git show --dirstat=files,cumulative 5614fc8aac81b227dab25f8fe341736c154d1145 web/
  git show --dirstat=cumulative,changes 5614fc8aac81b227dab25f8fe341736c154d1145 web/
  # To show detailed stats about lines added/deleted per file 
  git show --stat 5614fc8aac81b227dab25f8fe341736c154d1145 web/
  ```
