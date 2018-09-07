# UploadFilesFromFilesystem
PowerShell script to copy files and directories on a filesystem to a SharePoint document library.

[System.Reflection.Assembly]::LoadWithPartialName("Microsoft.SharePoint")
if((Get-PSSnapin | Where {$_.Name -eq "Microsoft.SharePoint.PowerShell"}) -eq $null) {
     Add-PSSnapin Microsoft.SharePoint.PowerShell;
}
#Site Collection where you want to upload files
$siteCollUrl = "http://sitecollectionurl" #Your Site Collection
#Document Library where you want to upload files
$libraryName = "Document Library Name" #Your Document Library Name
#Physical/Network location of files
$reportFilesLocation = "C:\Users\user\desktop\folder" #Top level of the file structure you want to replicate in SharePoint. Everything UNDER this directory will be copied to the document library.
$spSourceWeb = Get-SPWeb $siteCollUrl;
$spSourceList = $spSourceWeb.Lists[$libraryName];
if($spSourceList -eq $null)
{
    Write-Host "The Library $libraryName could not be found."
    return;
}   
$filelocation = ([System.IO.DirectoryInfo] (Get-Item $reportFilesLocation))
$files = $filelocation.GetFiles()
$subs =  $filelocation.GetDirectories()
foreach($file in $files)
{
    $fileStream = ([System.IO.FileInfo] (Get-Item $file.FullName)).OpenRead()
    #Add file
    $folder = $spSourceWeb.getfolder($libraryName)
    Write-Host "Copying file $file to $libraryName..."    
    Try{
        $spFile = $folder.Files.Add($folder.Url + "/" + $file.Name, [System.IO.Stream]$fileStream, $false)
    }
    Catch{
        Write-Host -ForegroundColor Green "The file " $file.name "already exists in this directory...skipping"
    }
    #Close file stream
    $fileStream.Close();   
}
Write-Host "Files have been uploaded to $libraryName."

#Get a collection of subdirectories
function UploadFilesToSubfolders($subs){
    foreach($sub in $subs){
        $url =($sub.FullName -split $reportFilesLocation,0,'simplematch')
        $url =$url.replace('\','/')
        $url = $url.trim(' ')
        $s = $siteCollUrl + "/" + $libraryName + $url.item(1).tostring()      
        $wfc = $spSourceWeb.getfolder($s).Exists
        if($wfc -eq $false){ 
                $subfolder = $spSourceWeb.getfolder($s)
                Write-Host "Creating $sub Folder..."
                $s.Trim($sub.name).trim("/")
                $i = $spSourceList.Items.Add($s.TrimEnd($sub.name).trim("/"),[Microsoft.SharePoint.SPFileSystemObjectType]"Folder",$sub.name)
                $i.Update()
                $fileloc = ([System.IO.DirectoryInfo] (Get-Item $sub.FullName)).GetFiles()
                $subs = ([System.IO.DirectoryInfo] (Get-Item $sub.FullName)).GetDirectories()
                UploadFilesToSubfolders($sub)              
                foreach($file in $fileloc){
                    #Open file
                    $f = $file.FullName                    
                    $fileStream = ([System.IO.FileInfo] (Get-Item $f)).OpenRead()
                    #Add file                   
                    $url =($file.FullName -split $reportFilesLocation,0,'simplematch')
                    $url =$url.Replace('\','/')
                    $folderName = $url.item(1).tostring()
                    $s = $siteCollUrl + "/" + $libraryName + "/" + $folderName.trim("/")
                    $subfolder = $spSourceWeb.getfolder($s)               
                    Write-Host "***Copying $file to $subfolder ...***"    
                    $spFile = $subfolder.Files.Add($subfolder.Url, [System.IO.Stream]$fileStream, $false)
                    #Close file stream
                    $fileStream.Close();                                        
                }           
        }
        else{
            Write-Host "Subdirectory" $sub.FullName "already exists."
            $subs = ([System.IO.DirectoryInfo] (Get-Item $sub.FullName)).GetDirectories()
            $fileloc = ([System.IO.DirectoryInfo] (Get-Item $sub.FullName)).GetFiles() 
            UploadFilesToSubfolders($subs)           
            Write-Host -ForegroundColor Cyan "Checking for new files..."
            foreach($file in $fileloc){
                    #Open file
                    $f = $file.FullName                    
                    $fileStream = ([System.IO.FileInfo] (Get-Item $f)).OpenRead()
                    #Add file                   
                    $url =($file.FullName -split $reportFilesLocation,0,'simplematch')
                    $url =$url.Replace('\','/')                 
                    $s = $siteCollUrl + "/" + $libraryName + "/" + $url.item(1).tostring()
                    $subfolder = $spSourceWeb.getfolder($s)
                    Try{
                        $spFile = $subfolder.Files.Add($subfolder.Url + "/" + $file.Name, [System.IO.Stream]$fileStream, $false)
                    }
                    Catch{
                        Write-Host -ForegroundColor Green "The file " $file.name "already exists in this directory...skipping"
                    }
                    #Close file stream
                    $fileStream.Close();
             } 
            
        }     
    }
}
UploadFilesToSubfolders($subs)
Write-Host -ForegroundColor Yellow "Done..."
$spSourceWeb.dispose();

