node {
    
    def TMP_FILE_REPO = "repo_info.log"
    def currentStage = ""
    def filterFiles  =".*.sql"
    
    def releaseTree = "#\u0411\u0414/MS SQL/FUIB_APP/"
    def releaseShareFolder = "\\\\DESKTOP-5EBVCN2\\share"
    def workspace = pwd()


    Map svnInfo = [:]
    Map sqlServers = ["uat": ["server":"192.168.1.52,1433",
                              "DB"    :"FUIB_CI", 
                              "credentialsId":"ed73e7c5-ea54-4c36-9033-3fd22c6ac894"], 
                      "lm" : ["server":"192.168.1.52,1433",
                              "DB"    :"FUIB_CI_LM", 
                              "credentialsId":"ed73e7c5-ea54-4c36-9033-3fd22c6ac894"]
                     ]
    try {

       // Mark the code 'checkout' stage....
       currentStage = 'Checkout'
       stage currentStage

       // Checkout code from repository
       checkout scm

       

        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'c1e3cb99-6582-4124-9a87-8889bce47618',
                     usernameVariable: 'SVN_USERNAME', passwordVariable: 'SVN_PASSWORD']]) {
          // get information about svn working copy
            def ret = bat (script:"svn info --no-auth-cache --trust-server-cert --non-interactive --username %SVN_USERNAME% --password %SVN_PASSWORD% >$TMP_FILE_REPO", returnStdout: true)
        // }
            svnInfo = parseSvnInfo(readFile(TMP_FILE_REPO))
            def revision = svnInfo.get("Revision")

          // get log text from last commit. It will be parsed for further work 
            ret = bat (script: "svn log --no-auth-cache --trust-server-cert --non-interactive --username %SVN_USERNAME% --password %SVN_PASSWORD% --verbose -r $revision >$TMP_FILE_REPO", returnStdout: true)
        }

        def parsed = parseSvnLogFiles(readFile(TMP_FILE_REPO).trim(),svnInfo.get("Relative URL"))

        svnInfo.put("LOG", parsed.get("description"))
        svnInfo.put("files", parsed.get("files") )
        svnInfo.put("tags", parsed.get("tags") )
        svnInfo.put("conf", parseTags(parsed.get("tags")) )
        println svnInfo
       

        // Mark the code 'Build' stage....
        currentStage = 'Build'
        stage currentStage
		
        def neededFiles = [:]


        for (String compileEnvCode : svnInfo.get("conf").get("compile")) {
        	println compileEnvCode
        	compileEnv = sqlServers.get(compileEnvCode)
        	if (compileEnv != null) {
		        // Run the sqlcmd
		        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: compileEnv.get("credentialsId"),
		                     usernameVariable: 'SQL_USERNAME', passwordVariable: 'SQL_PASSWORD']]) {
		       		for (String fileName : svnInfo.get("files")) {

                        println "Before compile '"+fileName+"'"
		       			def file = new File(workspace+"/"+fileName)
                        println file.getAbsolutePath()
		       			if (file.exists() && file.isFile() ) {
                            println "Before filter "+ svnInfo.get("conf").get("filter")[0]
		       				
 							if (fileFilter(file.getName(),svnInfo.get("conf").get("filter")[0])) {
 								if (!checkCodeRules(file.getName(),file.text))
 								{
 									throw new Exception("Not checked code rules")
 								}
                                def cmdLine = "sqlcmd -b -S "+compileEnv.get("server")+" -U %SQL_USERNAME% -P %SQL_PASSWORD% -d " +compileEnv.get("DB")+ " -i "+fileName
                                println cmdLine

 								bat cmdLine

                                neededFiles.put(fileName,file.getName())
							} else {
								println "miss "+file.getName()
							}
		       			}
		       		}

		        	
				}
			}
		}

        currentStage = 'Release Prepare'
        stage currentStage
        def releaseData = [:]
        def items = []

        String dateStamp = (new Date()).format( 'yyyy.MM.dd hh.mm' )
        releaseData.put("dstFolder",releaseShareFolder)
        releaseData.put("dateStamp",dateStamp)
        def releaseId = svnInfo.get("conf").get("ID")
        if (releaseId == null) {
            releaseId = "SVNSQLREV"+svnInfo.get("Last Changed Rev")
        }
        releaseData.put("ID",releaseId)
        releaseData.put("subject",svnInfo.get("conf").get("subject"))
        println dateStamp

        def putDescription = true
        for (fileName in neededFiles) {
            def item = [:]
            def code = removeExtension(fileName.value)
            item.put("copyFrom",workspace+"/"+fileName.key)
            item.put("copyToFolder",releaseTree+code+"/"+dateStamp)
            item.put("newFileName",fileName.value)
            item.put("code",code)
            if (putDescription) {
                item.put("description",svnInfo.get("LOG"))
                putDescription = false;
            }
            items.add(item)
        }
        releaseData.put ("items",items)
        releaseData.put ("mail",findEmail (svnInfo.get("Last Changed Author").trim()))

        prepareFuibRelease(releaseData)

        


    } catch(Exception e) { 
        def toEmail = findEmail (svnInfo.get("Last Changed Author").trim())
        println toEmail

        emailext attachLog: true, 
                      body: '<h2> <a href="'+env.BUILD_URL+'">Job "'+env.JOB_NAME+'" ('+env.BUILD_NUMBER+') failed </> </h2>'  , 
               compressLog: true,
                   subject: 'JENKINS CI Exception. Stage: '+currentStage+'. SVN: '+svnInfo.get("Relative URL") , 
                        to: toEmail
        throw e;
    }

}

def parseSvnInfo (body) {
    def result = [:]
        
    for (it in body.split("\n")) {
        p = it.indexOf(":", 0)
        println ([it , p])
        if (p>0) {
            result.put(it.substring(0,p) , it.substring(p+1,it.length()).trim())
        }
    }
    return result
}
/*
SAMPLE of console out
g:\Projects\java\_svn\web_app\branches\FEATURE-1>svn log --verbose -r 108
------------------------------------------------------------------------
r108 | Miha | 2016-07-11 16:12:44 +0300 (Пн, 11 июл 2016) | 1 line
Changed paths:
   M /Java/web_app/branches/FEATURE-1/Jenkinsfile

mask fuib
------------------------------------------------------------------------

Relative URL: ^/Java/web_app/branches/FEATURE-1

*/


def parseSvnLogFiles (body,relativeUrl) {
    def result = [:]
    def files = []
    def tags = []
    relativeUrl=relativeUrl.substring(1)+"/";
    def filesLine = false;
    def logLine   = false;
    def line = "";
    def logLineStr=""
    int i = 1;
    println "RelativeURL :" + relativeUrl
    for (it in body.split("\n")) {
        println "Current Line :" + it
    	if (it.trim().length()==0) {
    		filesLine=false;
    		logLine = true;
    	}
    	if (i>3 && !logLine && !filesLine) {
    		filesLine=true;
    	}

    	if (filesLine) {
    		line=it.substring(5)
    		if (line.substring(0,relativeUrl.length()).equals(relativeUrl)) {
                if (relativeUrl.length()<line.length()) {
                   println "FILE:"+line.substring(relativeUrl.length())
	               files.add(line.substring(relativeUrl.length()).trim());
                }
        	}
    	}

    	if (logLine) {
    		logLineStr+=it
    	}

        ++i;
    }

    def value = logLineStr
    boolean foundTag=false;
    StringBuffer sbTag = new StringBuffer("");
    StringBuilder sbComment = new StringBuilder("");
    for (i=0;i<value.length();i++){
        if (value.charAt(i)=='#') {
            if (!foundTag) {
                foundTag=true;
            } else {
                tags.add(sbTag.toString());
                sbTag=new StringBuffer("");
                foundTag=false;
            }
        } else {
            if (foundTag) {
                sbTag.append(value.charAt(i));
            } else {
                sbComment.append(value.charAt(i));
            }
        }
    }
    result.put("files",files)
    result.put("tags",tags)
    result.put("description",sbComment.toString())


    return result
}

def parseTags(tags) {
	def result = [:]
	for (String tag : tags) {
    	def parsed = tag.split("!")
    	if (parsed[0].equals("compile")||parsed[0].equals("c")) {
    		def tmp = []
    		for(int i=1;i<parsed.length;i++) {
    			tmp.add(parsed[i])
    		}
    		result.put("compile",tmp)
    	} else
    	if (parsed[0].equals("filter")||parsed[0].equals("f")) {
    		result.put("filter",parsed[1])
    	} else
    	if (parsed[0].equals("release")||parsed[0].equals("r")) {
    		int tmpI = 1
    		if (parsed[tmpI].substring(0,4).equals("rev:")) {
    			result.put("revisions",parsed[tmpI])
    			++tmpI;
    		}
    		if (parsed[tmpI].substring(0,3).equals("ID:")) {
    			result.put("ID",parsed[tmpI].substring(3))
    			++tmpI;
    		}
    	}
    	if (parsed[0].equals("mail")||parsed[0].equals("m")) {
    		result.put("mail",parsed[1])
    	}
        if (parsed[0].equals("subject")||parsed[0].equals("s")) {
            result.put("subject",parsed[1])
        }
	}
	// default
	if (result.get("compile")==null) {
		result.put("compile",["uat"])
	}
	if (result.get("filter")==null) {
		result.put("filter",[".*\\.sql"])
	}
    if (result.get("subject")==null) {
        result.put("subject","Релиз FUIB_CI_LM")
    }
	return result
}

def checkCodeRules(fileName, fileBody) {
	return true
}

def findEmail(author) {
    def emails = ["Miha":"mmihaylovich@ya.ru"]
    def email_suffix = "@xxxx.com"

    def email = emails.get(author)

    if (email == null) {
        return author+email_suffix
    } else {
        return email
    }

}


def removeExtension(String filename) {
    if (filename == null) {
        return null;
    }
    int index = indexOfExtension(filename);
    if (index == -1) {
        return filename;
    } else {
        return filename.substring(0, index);
    }
}

def indexOfExtension(String filename) {
    if (filename == null) {
        return -1;
    }
    int extensionPos = filename.lastIndexOf('.');
    int lastSeparator = indexOfLastSeparator(filename);
    return (lastSeparator > extensionPos ? -1 : extensionPos);
}



def indexOfLastSeparator(String filename) {
    if (filename == null) {
        return -1;
    }
    int lastUnixPos = filename.lastIndexOf('/');
    int lastWindowsPos = filename.lastIndexOf('\\');
    return Math.max(lastUnixPos, lastWindowsPos);
}

def prepareFuibRelease(releaseData) {
    int count = 0
    def workspace = pwd()
    def folder = workspace+"/release"
    if ((new File(folder)).exists()) {
       bat (script:"RD /S /Q "+workspace+"\\release", returnStdout: true)
    }
    if ((new File(workspace+"/release_out")).exists()) {
       bat (script:"RD /S /Q "+workspace+"\\release_out", returnStdout: true)
    }
    folder = folder + "/"

    for (item in releaseData.get("items")){

        // def folderName = new String( (folder+item.get("copyToFolder")).getBytes("Cp1251"))
        def folderName = folder+item.get("copyToFolder")
        println "folderName " + folderName
        def file = new File ( folderName ) 
        file.mkdirs()
        println "src: "+item.get("copyFrom")
        println "dst: "+folderName+"/"+item.get("newFileName")
        def src = new File(item.get("copyFrom"))
        def dst = new File(folderName+"/"+item.get("newFileName"))
        dst << src.bytes
        // java.nio.file.Files.copy(item.get("copyFrom"), folderName+"/"+item.get("newFileName"), java.nio.file.StandardCopyOption.REPLACE_EXISTING);
        ++count;
    }
    if (count==0) return

    def zipFile = releaseData.get("dateStamp")+" "+releaseData.get("ID")+".zip"
    def zipFullFileName = workspace+"\\release_out\\"+zipFile

    bat (script:"start /wait 7z a -tzip \""+zipFullFileName+"\" \""+folder+"*\"")


    def TMP_FILE_MD5 = workspace+"\\release_out\\md5_info.log"
    bat (script:"fciv -md5 \""+zipFullFileName+"\"  >$TMP_FILE_MD5", returnStdout: true)

        // def md5 = readFile(TMP_FILE_MD5).trim().split("\n")[3].substring(0,32)
    def md5 = (new File(TMP_FILE_MD5)).text
    println "MD5:" md5 


    bat (script:"copy  \""+zipFullFileName +"\" "+releaseData.get("dstFolder")+"\"")

    // def htmlTable = ""
    // for (item in releaseData.get("items")){
    //     htmlTable+="<tr><td>"+item.get("newFileName")+"</td><td>"+item.get("code")+"</td><td>"+item.get("description")+"</td></tr>"
    // }
    // def textBody  = '''
    //     '''+  releaseData.get("dstFolder") +" md5("+zipFile+")="+md5 +'''
    //     <table> '''+         htmlTable+'''
    //     </table>
    // '''
    //     emailext attachLog: true, 
    //                   body: textBody, 
    //            compressLog: true,
    //                subject: 'RELEASE SQL', 
    //                     to: releaseData.get("mail")

    //md5
    /*
    g:\win10pc\temp\fciv>fciv -md5 ReadMe.txt
    //
    // File Checksum Integrity Verifier version 2.05.
    //
    79ac8d043dc8739f661c45cc33fc07ac readme.txt
    */
    
}

// println parseSvnInfo(new File('svninfo.txt').text)

@NonCPS
def fileFilter(text,filter) {
  def matcher = text =~ filter
  matcher
}
