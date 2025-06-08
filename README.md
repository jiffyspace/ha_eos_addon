# Home Assistant EOS addons repository

Home Assistant EOS addons

[![Open your Home Assistant instance and show the add add-on repository dialog with a specific repository URL pre-filled.](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2FDuetting%2Fha_eos_addon)

## Add-ons

This repository contains the following add-ons

- **[EOS add-on](./eos/README.md)**
  This is just a home assistant add-on wrapper for the great work here: <https://github.com/Akkudoktor-EOS/EOS>

  ![Supports aarch64 Architecture][aarch64-shield]
  ![Supports amd64 Architecture][amd64-shield]

- **[EOS local build add-on](./eos_local_build/README.md)**
  This is just a home assistant add-on wrapper which will be built locally for the great work here: <https://github.com/Akkudoktor-EOS/EOS>

  ![Supports aarch64 Architecture][aarch64-shield]
  ![Supports amd64 Architecture][amd64-shield]

<!--

Notes to developers after forking or using the github template feature:
- While developing comment out the 'image' key from 'example/config.yaml' to make the supervisor build the addon
  - Remember to put this back when pushing up your changes.
- When you merge to the 'main' branch of your repository a new build will be triggered.
  - Make sure you adjust the 'version' key in 'example/config.yaml' when you do that.
  - Make sure you update 'example/CHANGELOG.md' when you do that.
  - The first time this runs you might need to adjust the image configuration on github container registry to make it public
  - You may also need to adjust the github Actions configuration (Settings > Actions > General > Workflow > Read & Write)
- Adjust the 'image' key in 'example/config.yaml' so it points to your username instead of 'home-assistant'.
  - This is where the build images will be published to.
- Rename the example directory.
  - The 'slug' key in 'example/config.yaml' should match the directory name.
- Adjust all keys/url's that points to 'home-assistant' to now point to your user/fork.
- Share your repository on the forums https://community.home-assistant.io/c/projects/9
- Do awesome stuff!
 -->

## Contribution

Currently the most work in this repo is updating the EOS source code and check if the changes still work with the documentation.
To do this the following steps are important:

- update eos: `git submodule update --remote --recursive`
- Run the vscode task `Start Home Assistant`
  - this takes up to 10 mins
  - if you see for a while on the onboading screen an issue with DNS just wait longer
- After setting up the devloperment homeassitant instance install the Addon `EOS add-on`
- Start the addon
- After starting go to the folder `tests`
- Run the request in `prognose.http`
- The result should be a 200 OK and should contain anything that is refered in the `DOCS.md`
