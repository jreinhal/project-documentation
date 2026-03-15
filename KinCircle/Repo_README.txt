manage-reference-repos:

Run this if you have added new repositories or just want to ensure everything is configured:
powershell
cd D:\Projects\reference-repos
.\manage-reference-repos.ps1 -Scan

Run this to see a status report of all repos and their tasks:
powershell
.\manage-reference-repos.ps1 -List


If you want the script to clone a repo for you and immediately set it up:
powershell
.\manage-reference-repos.ps1 -Add "https://github.com/StartDD/Awesome-Agentic-Reasoning"