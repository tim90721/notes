- [Clone](#clone)
  - [Reference](#reference)
- [Tag](#tag)
  - [Reference](#reference-1)

# Clone
- Clone a repository
  ```bash
  git clone ${URL}
  ```
- Clone a repository and its submodules
  ```bash
  git clone --recurse-submodules -j8 ${URL}
  ```
  > [!NOTE] -j8 is optional
- Fetch and download the repository's submodules if it is already cloned
  ```bash
  git submodule update --init --recursive
  ```
## Reference
- https://stackoverflow.com/questions/3796927/how-do-i-git-clone-a-repo-including-its-submodules

# Tag
- Check list of tags
  ```bash
  git tag -l
  ```
- Checkout specific tag
  ```bash
  git checkout tags/<tag_name>
  ```
## Reference
- https://stackoverflow.com/questions/791959/download-a-specific-tag-with-git