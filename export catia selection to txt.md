```vb
Sub CATMain()

    Dim sPath
    Dim i
    Dim oSelection                               'As Selection
    Dim oSelectedElement                         'As SelectedElement
    Dim sElementName                             'As String
    Dim sElementPath                             'As String
    Dim sInput
    Dim sType
    Dim sName
    Dim objDictionary
    Dim key 'As Variant
    Dim totalElementCount

    Set objDictionary = CreateObject("Scripting.Dictionary")

    sPath = CATIA.FileSelectionBox("Export Selection to .txt, append mode", "*.txt", CatFileSelectionModeSave)
   
    If sPath = "" Then
        MsgBox "Nothing is Selected, bye!", vbCritical
        Exit Sub
    End If

    '  get Parameters of root assembly.
    Set prdAssembly = CATIA.ActiveDocument.Product
    Set objAssemblyParams = prdAssembly.Parameters
   
    Set oSelection = CATIA.ActiveDocument.Selection
 
    oSelection.Clear
 
	sInput = InputBox("Enter search string:", "Export CATIA Selection to text file", "name=*,all") 
	
    oSelection.Search sInput

    Set fs = CreateObject("Scripting.FileSystemObject")

    Set f = fs.OpenTextFile(sPath, 8, 0)
    
    ' since it is append mode, need to tell who and when did this.
    strUser = CreateObject("WScript.Network").UserName

    sLine = strUser & "-" & Date & "-" & Time & vbCrLf
    f.Write sLine
   
    For i = 1 To oSelection.count
      
        Set oSelectedElement = oSelection.Item(i).value
        
        sElementPath = removeComma(objAssemblyParams.GetNameToUseInRelation(oSelectedElement))     
 
		sType = TypeName(oSelectedElement)

		sName = removeComma(oSelectedElement.name)

		sLine = sType + "," + sName + "," + sElementPath + vbCrLf

		f.Write sLine

		' add summary information to Dictionary
		If objDictionary.Exists(sType) Then 
			objDictionary(sType) = objDictionary(sType) + 1
		else
			objDictionary(sType) = 1
		end if 
    Next

    f.write "Summary" & vbCrLf
    ' write summary infomation to file.
    For Each key In objDictionary.Keys
        f.Write key & "," & objDictionary(key)  & vbCrLf
        totalElementCount = totalElementCount + objDictionary(key) 
    Next
    oSelection.Clear
    f.write vbCrLf
    f.Close

    MsgBox totalElementCount & " Elements Exported!"

End Sub

' remove comma since comma is used in delimmiter.
function removeComma(str)
    removeComma = replace(str,","," ")
end function

```
