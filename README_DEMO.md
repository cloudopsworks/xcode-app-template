# I am Groot
This is my first iOS project and it will serve as a playgroud to learn iOS programming and demonstrate mobile app security and privacy testing as well as DevOps automation with GitHub Actions.

1. Code Setup
   If you want to follow along with my “hello world” iOS app, make sure you follow the code setup from the previous post to download the source code to your computer. Obviously just use your iOS app if you have one in mind! Also, the post assumes you have already exported the app archive at least once from Xcode.

2. Configuration inputs
   As discussed in the post on building from command line, there are a number of files and parameter you have to manually handle vs. a Xcode managed workflow. For the GitHub Action, we will need:

Signing certificate (and a strong password to protect it)
Mobile provisioning profile for the app
Keychain password for the build machine
ExportOptions.plist
There are plenty of blogs that step you through the process of creating the first three items manually in Apple’s Developer Portal so feel free to check them out. But I’m lazy and realized that Xcode already did all of this for me so I could simple reuse what was already configured (ymmv)!

To simplify access to these inputs, please all the files in a single folders, e.g.:

$ mkdir -p ~/spfexpert/ios-deploy && cd $_
Copy
2.1 Signing certificate#
Since you’ve already built your app in Xcode, you can simple run Keychain Access (just do a Spotlight Search by hitting ⌘-Space) and do the following:

Select login on left “Default Keychains” panel
Select “My Certificates” from horizontal navigation
Right-click your certificate and select “Export…”
Choose your location (~/spfexpert/ios-deploy) and file name (defaults to Certificates.p12)
Enter strong password to protect the certificate
Apple Keychain Access - My Certificates

Tip: finding the correct signing certificate and provisioning profile#
If you’re just starting out with iOS development, it’s likely you might have just a single certificate. As you can tell above, I have two certificates and while working on the Github Action, I chose the wrong certificate multiple times! One way to figure out the appropriate certificate to build the app from the command line on your macOS and look for the CodeSign output in logs, e.g.:

CodeSign /Users/hiro/Library/Developer/Xcode/DerivedData/I_am_Groot-gfxpkzbpazztxzabhewsezcfrozd/Build/Products/Debug-iphoneos/I\ am\ Groot.app (in target 'I am Groot' from project 'I am Groot')
cd /Users/hiro/spfexpert/iamgroot

    Signing Identity:     "Apple Development: Andrew Hoog (ZJN98QQ2HM)"
    Provisioning Profile: "iOS Team Provisioning Profile: *"
                          (1d0e8da1-9eba-41c7-a308-931ba380c3b0)
Copy
Once I saw this, it was obvious that I needed the first certificate in Keychain Access!

2.2 Mobile provisioning profile for the app#
The next item we need is the mobile provisioning profile. Using the little hack of looking at the CodeSign output from a command line build, we can see which provisioning profile Xcode used. The provisioning profiles are stored on your macOS at ~/Library/MobileDevice/Provisioning\ Profiles so you can simple copy the appropriate files as follows:

$ cp ~/Library/MobileDevice/Provisioning\ Profiles/1d0e8da1-9eba-41c7-a308-931ba380c3b0.mobileprovision ~/spfexpert/ios-deploy/
Copy
2.3 Keychain password for the build machine#
The last step is just to create a password you will use to setup and add data to the GitHub Action macOS runner’s Keychain. You can use whatever technique you like but I’ve been a huge fan of of zx2c4’s password-store for many years now and generate the password as follows:

hiro@sophon:~/spfexpert|⇒  pass generate temp/temp 18
An entry already exists for temp/temp. Overwrite it? [y/N] y
[master 7159eaa] Add generated password for temp/temp.
1 file changed, 0 insertions(+), 0 deletions(-)
rewrite temp/temp.gpg (100%)
The generated password for temp/temp is:
w*cp,k.To~A^-g@MK-
Copy
Since there are temporary passwords for me, I just overwrite it.

2.4 ExportOptions.plist#
This is a required configuration that you could create manually but is saved by Xcode when you archive an app. I’ve already documented how to snag this file so review that post for more details. So assuming you’ve exported your iOS app archive to ~/spfexpert/iamgroot/build, you could copy the ExportOptions.plist as follows:

$ cp ~/spfexpert/iamgroot/build/ExportOptions.plist ~/spfexpert/ios-deploy
Copy
3. Repository secrets setup
   Now we have all the information we need to configure the GitHub Action to export our iOS app archive. The great news is that GitHub has an fantastic doc that gives us the exact workflow steps we need to follow.

To protect the sensitive configuration data, the values from above will be stored as repository secrets which will require us to serialize the data using base64. In other blogs, some folks take the additional steps of encrypting the config data with gpg however I do not believe this provides sufficient additional security. But certainly feel free to add the additional layer if you like but recognize that the passcode for gpg will also be stored as a repository secret so if that layer is every compromised, it won’t provide any additional protection.

We’ll pipe the output of base64 into pbcopy so it’s simple to then paste the resulting base64 data into your GitHub repository secrets.

First, in your repo on github.com, navigate to Settings -> Secrets -> Actions (e.g. https://github.com/ahoog42/iamgroot/settings/secrets/actions) and then click the green “New repository secret” button:

Repository secrets config on github.comFrom here, we’ll create new 5 repository secrets with the Name (from the heading) and then Secret as follows:

3.1 BUILD_CERTIFICATE_BASE64#
Base64 the signing certificate and pipe it to the clipboard:

$ base64 -i Certificates.p12| pbcopy
Copy
and then paste the clipboard contents (⌘-V) into the textbox labeled Secret.

3.2 P12_PASSWORD#
Copy the password you used to export your signing certificate to the clipboard, e.g.

$ pass -c personal/apple/certs/XW66E6M5N4
Copy
and then paste the clipboard contents (⌘-V) into the textbox labeled Secret.

3.3 BUILD_PROVISION_PROFILE_BASE64#
Base64 the mobile provisioning profile and pipe it to the clipboard:

$ base64 -i 1d0e8da1-9eba-41c7-a308-931ba380c3b0.mobileprovision| pbcopy
Copy
and then paste the clipboard contents (⌘-V) into the textbox labeled Secret.

3.4 KEYCHAIN_PASSWORD#
Copy the password you created for macOS runner’s Keychain to the clipboard, e.g.

$ pass -c temp/temp
Copy
and then paste the clipboard contents (⌘-V) into the textbox labeled Secret.

3.5 EXPORT_OPTIONS_PLIST#
And finally base64 the ExportOptions.plist file and pipe it to the clipboard:

$ base64 -i ExportOptions.plist| pbcopy
Copy
and then paste the clipboard contents (⌘-V) into the textbox labeled Secret.