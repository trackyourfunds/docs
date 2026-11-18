# Telerik license load into Visual Studio on your machine

- Log into Telerik with your Telerik licensed ID ([tyfadmin@trackyourfunds.com](mailto:tyfadmin@trackyourfunds.com)). Password stored securely in 1Password.
- Go to ‘Your Licenses’ under the profile logo

![](Telerik_Images/image_001.png)

- On the Licenses Page, click on View License Keys

![](Telerik_Images/image_002.png)

- Click on Manual Setup & then Set Up License Key

![](Telerik_Images/image_003.png)

- Click on Download License Key

![](Telerik_Images/image_004.png)

- On your workstation browse to ‘Users/username/AppData/Roaming/Telerik & copy the Telerik-license.txt file you just downloaded into that folder. If a folder does not exist under Roaming, create a folder called Telerik.

![](Telerik_Images/image_005.png)

- In Visual Studio do a build TYF which forces a nuget restore and it should show it found a license.

![](Telerik_Images/image_006.png)

