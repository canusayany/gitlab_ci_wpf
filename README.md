# 使用Gitlab配置.net8WPF的CICD
  大致分成三部分.1.配置Runner.2.配置.gitlab-ci.yml.3.项目本身配置.
  ## 1.配置Runner
  在项目设置Settings->CI/CD->Runners->new PRoject Runner->把提供的注册信息填入runner在的文件夹的powershell->运行->如果需要打包不带tags的提交需要编辑runner的设置->勾选Run untagged jobs @ Indicates whether this runner can pick jobs without tags.
  如果使用powershell记得将runner的配置文件内的 shell = "powershell"
  ## 2.配置.gitlab-ci.yml
  项目配置build->Pipeline editor->选择分支->编辑文本->提交更新.
  ```yaml
  variables:
  Dir_path: 'D:\LiquidStudio\publish\'
  TempDir_path: 'D:\Temp'
  Zip_path: 'D:\LiquidStudio\publish\Zip\'
  zipFileName: ''
stages:          # List of stages for jobs, and their order of execution
  - deploy
  - packet
  
deploy:      # This job runs in the deploy stage.
  stage: deploy  # It only runs when *both* jobs in the test stage complete successfully.
  only: 
    - dev
  script:
    - echo "Deploying application..."
    #- dotnet nuget locals all --clear
    - dotnet restore -s "http://192.168.20.79:8080/v3/index.json"
    #- dotnet nuget locals all --clear
    - echo "restore over"
    #检查发布文件夹是否存在,如果存在就删除
    - if (Test-Path $TempDir_path) { Remove-Item $TempDir_path -Recurse -Force }
    #发布sln文件所在的文件夹下\src\BR.PC.App.LiquidStudio\BR.PC.App.LiquidStudio.csproj到服务器 框架依赖 目标运行时win-x64 发布到临时文件夹下
    - dotnet publish .\src\BR.PC.App.LiquidStudio\BR.PC.App.LiquidStudio.csproj -c Release -o $TempDir_path\ -r win-x64 -p:DebugType=None --no-restore
    #将sln文件所在的文件夹下的refLib文件夹内部的WebView2Loader.dll拷贝到发布目录下
    - Copy-Item -Path .\refLib\win\WebView2Loader.dll -Destination $TempDir_path -Force
    # 拷贝解决方案所在的文件夹下的wwwroot文件夹到发布目录下
    - Copy-Item -Path .\wwwroot -Destination $TempDir_path'\wwwroot' -Recurse -Force
    - echo "Application successfully deployed."
    - $time1 = Get-Date -Format "yyyyMMddHHmmss"
    - echo $time1

packet:     
  stage: packet  
  #needs: [deploy]
  dependencies: 
    - deploy
  only: 
    - dev
  script:
    - echo "packet the code..."
    - echo $TempDir_path
    - $time2 = Get-Date -Format "yyyyMMddHHmmss"
    - echo $time2
    #获取版本号编译后的.net程序集
    - $version = (Get-Item $TempDir_path\BR.PC.App.LiquidStudio.dll).VersionInfo.ProductVersion
    - echo $version
    #截取版本号前两节
    - $version = $version.Substring(0, $version.IndexOf('.', $version.IndexOf('.') + 1))
    - echo $version
    #将当前时间存储到变量中
    - $time = Get-Date -Format "yyyyMMddHHmmss"
    - $zipFileName= 'LiquidStudio ' + $version + '.' + $time
    #Dir_path后边加上版本号
    - $Dir_path = $Dir_path +'LiquidStudio '+ $version+'.'+$time
    - echo $Dir_path
    #创建文件夹$Dir_path
    - New-Item -Path $Dir_path -ItemType Directory
    #将TempDir_path下的所有文件移动到$Dir_path文件夹下
    - Move-Item -Path $TempDir_path\* -Destination $Dir_path
    #如果$Zip_path不存在就创建
    - if (!(Test-Path $Zip_path)) { New-Item -Path $Zip_path -ItemType Directory }
    #将$Dir_path文件夹下的所有文件压缩到Zip_path文件夹下的'LiquidStudio '+ $version+'.'+$time.zip文件中
    - Compress-Archive -Path $Dir_path\* -DestinationPath ($Zip_path + 'LiquidStudio ' + $version + '.' + $time + '.zip')
    #在当前目录下创建Temp文件夹(如果存在就不创建文件夹但是要删除内部的所有内容),并将TempDir_path内的所有文件拷贝过来
    - $ArtifactsPath='LiquidStudio '+ $version+'.'+$time
    - if (Test-Path .\$ArtifactsPath) { Remove-Item .\$ArtifactsPath -Recurse -Force }
    - New-Item -Path .\$ArtifactsPath -ItemType Directory
    - Copy-Item -Path "$Dir_path\*" -Destination ".\$ArtifactsPath" -Recurse -Force
    - echo "packet complete."
  artifacts:
    reports:
      dotenv: build.env
    when: on_success
    untracked: true
    paths:
      - $ArtifactsPath/
      #定义artifacts的名字
    name: LiquidStudio_$CI_COMMIT_TIMESTAMP
after_script:
  - echo "deploy complete!!!!!!"
```
## 3.配置项目
### 修改项目文件
```xml
<PropertyGroup>
	<OutputType>WinExe</OutputType>
	<TargetFramework>net8.0-windows</TargetFramework>
	<RuntimeIdentifiers>win-x64</RuntimeIdentifiers>
	<Nullable>enable</Nullable>
	<ImplicitUsings>enable</ImplicitUsings>
	<UseWPF>true</UseWPF>
	<RestoreSources>$(RestoreSources);$(PaketSources)</RestoreSources>
	<RestoreSources Condition="'$(DotNetBuildOffline)' == 'true'">$(PaketSources)</RestoreSources>
	<AssemblyVersion>1.0.8783.3351</AssemblyVersion>
	<FileVersion>1.0.8783.3351</FileVersion>
</PropertyGroup>

```
### 如果有编译事件
可以参考,有的宏是无效的
``` xml
 <Target Name="PostBuild" AfterTargets="PostBuildEvent">
  <Exec Command="call &quot;../../.copyWwwrootToLiquidStudio.bat&quot; ..\.. $(TargetDir)&#xD;&#xA;call &quot;../../.copyWinToLiquidStudio.bat&quot; ..\.. $(TargetDir)" />
 </Target>
```
