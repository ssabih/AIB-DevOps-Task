<p> 
A common customer use case which I come across is “How to keep my WVD master image up to date” especially when it comes to windows update.
Azure Image Builder DevOps Task simplifies that by providing you with DevOps Task that can run on schedule and keep your master image up to date. You can surely do a lot of things with Azure image builder but in this article, we will focus on how to automate windows update for our WVD master image and then replicate it to multiple regions. 
At the time of writing this Azure image builder is in preview and available in the following regions. 
<ul>
<li>East US</li>
<li>East US 2</li>
<li>West Central US</li>
<li>West US</li>
<li>West US 2</li>
<li>North Europe</li>
<li>West Europe</li>
</ul>
This means that the build will run only in these regions. Once the image is ready, it can be replicated to other regions through SIG. In my case it will be Australia East. You can also use AIB to distribute the image to other regions but the operation can take very long time depending upon the location and may cause the DevOps task to fail. I faced this issue, therefore I decided to just use shared image gallery for replication.
</P>
<P>Let’s get started.</P>

### Objectives
In this example we will perform simple tasks needed to keep a Windows-10 image up to date with windows update.
<ol>
<li>Find the latest Image from the Shared Image Gallery (SIG).</li>
<li>Perform windows update on this image.</li>
<li>Distribute it as a new version of the image in the same SIG.</li>
<li>Replicate it to multiple Azure regions.</li>
</ol>

### Pre-requisites
<ol>
<li>Create a resource group for AIB. I created mine in West US as its one of the supported regions.</li>
<li> <a href="https://github.com/danielsollondon/azvmimagebuilder/blob/master/quickquickstarts/0_Creating_a_Custom_Windows_Managed_Image/readme.md#step-1--enable-prereqs-1">Register AIB</a>. You can do it through Cloud Shell.</li>
<li>Create a <a href="https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal">User assigned managed identity </a></li>
<li>Give permissions to “User assigned managed identity” created in step 1. For testing purpose, I gave it contributor to subscription. For details on specific permissions read <a href="https://github.com/danielsollondon/azvmimagebuilder/blob/master/aibPermissions.md#azure-powershell-examples">this post. </a></li>
<li>Create a Shared Image Gallery in the same region as step 1. For me it was West US.</li>
<li>Place your master image in the Shared Image Gallery.</li>
</ol>

### Building AIB DevOps Task
<ul>
<li>Create DevOps project.</li>
<img src="AIB_files/image001.png" width="200" height="100">

<li>Give it a name and click create.</li>
<img src="AIB_files/image003.png" width="400" height="400">


<li>Create a Release Pipeline by clicking <b>“New Pipeline”</b>.</li>
<img src="AIB_files/image005.png">

<li>Under <b>“Select a template”</b>, click on <b>“Empty Job”</b>.</li>
<img src="AIB_files/image007.png">

<li>Name your <b>Stage</b>.</li>
<img src="AIB_files/image009.png">

<li>Click the task in <b>“Build Image”</b> stage.</li>
<img src="AIB_files/image011.png">

<li>Add a task of type <b>“Azure PowerShell”</b>.</li>
<img src="AIB_files/image013.png">

<li>Set the values in the following script, replace all the placeholder values in caps with your values. We will use this script in the next step.</li>
</ul>
```markdown
$currentAzContext = Get-AzContext
$imageResourceGroup="ROSOURCE-GROUP"
$subscriptionID=$currentAzContext.Subscription.Id
$sigGalleryName= "SIG-GALLERY-NAME"
$imageDefName ="IMAGE-DEF-NAME"
Install-Module -Name Az.ManagedServiceIdentity -Confirm:$false -Force
$idenityObject=$(Get-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup | Where-Object {$_.Name -Match "AIB-IDENTITY*"})
$idenityNameResourceId=$idenityObject.Id
$idenityNamePrincipalId=$idenityObject.PrincipalId
$idenityName=$idenityObject.Name
$getAllImageVersions=$(Get-AzGalleryImageVersion -ResourceGroupName $imageResourceGroup  -GalleryName $sigGalleryName -GalleryImageDefinitionName $imageDefName)
$versionPubList=$($getAllImageVersions | Select-Object -Property Name -ExpandProperty PublishingProfile)
$sortedVersionList=$($versionPubList | Select-Object Name, PublishedDate | Sort-Object PublishedDate -Descending | Select-Object Name -First 1)
$sigDefImgVersionId=$(Get-AzGalleryImageVersion -ResourceGroupName $imageResourceGroup  -GalleryName $sigGalleryName -GalleryImageDefinitionName $imageDefName -Name $sortedVersionList.name).Id
echo "##vso[task.setvariable variable=latestversionid]$sigDefImgVersionId"
```
<ul>
<li>Set the <b>Display Name, Subscription</b>. Copy the script created in the previous step in the <b>Inline Script</b> section and set the <b>PowerShell version</b>, you can use simply use the <b>latest installed version</b>.</li>
<img src="AIB_files/image015.png">

</ul>

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```


For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/ssabih/Pages/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
