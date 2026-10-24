/*	load("D:/Temp/My_Skill/HCIS_ErrorPadUsageTracer.il") 
	encrypt("D:/Temp/My_Skill/HCIS_ErrorPadUsageTracer.il" "D:/Temp/My_Skill/Henry_HCIS_ErrorPadUsageTracer.il") 
	2026/10/22		V0.1B1		检查Drill有设定drill type, 但是drill size为0的padstack.
	.							另外, 检查此Padstack在哪些symbol中有用到.
	
	
*/ 

axlCmdRegister("ErrPadUsageTracer" 'HCIS_ErrPadUsageTracer ?cmdType "general")

; 0. 主函數 
procedure( HCIS_ErrPadUsageTracer()
let(( Path0 DirS DirN PathN formfile) 
	glb_checkPaths=nil
	foreach(Path0 list("K:/pads" "K:/KORE_R9IA/pads")  ; 42930個 + 103個 
		DirS = remove(".." remove("." getDirFiles(Path0))) 
		foreach(DirN DirS
			PathN = sprintf(nil "%s/%s" Path0 DirN)
			glb_checkPaths=cons(PathN glb_checkPaths)
		) 
	) 
	glb_startDir="."
	formfile = HCIS_ErrPadSym_CrtFormFile()
	Form = axlFormCreate( (gensym) formfile nil 'HCIS_ErrPadSym_CallBack nil)
	deleteFile(formfile)
	axlFormSetField(Form "path_list" glb_checkPaths)
	axlFormSetFieldEditable(Form "btn_del" nil)
	axlFormSetInfo(Form "process_info" "have drill type and drill size = 0.0, check padstack and used symbols") 
	axlFormColorize(Form "process_info" 'text 'blue)
	axlFormDisplay( Form ) 
))

; 1. Form文件 
procedure( HCIS_ErrPadSym_CrtFormFile()
let((tab vec op ls filename a n)
	tab=makeTable("" nil) n=0
	when(t
	tab[n++]=sprintf(nil "FILE_TYPE=FORM_DEFN VERSION=2")
	tab[n++]=sprintf(nil "FORM")
	tab[n++]=sprintf(nil "FIXED")
	tab[n++]=sprintf(nil "PORT 50 100")
	tab[n++]=sprintf(nil "HEADER %L" "Padstack Drill Tol Check")
	tab[n++]=sprintf(nil "")
	tab[n++]=sprintf(nil "TILE")
	tab[n++]=sprintf(nil "")
	tab[n++]=sprintf(nil "TEXT %L" "Path for check:")
	tab[n++]=sprintf(nil "TLOC 1 1")
	tab[n++]=sprintf(nil "ENDTEXT")
	tab[n++]=sprintf(nil "")
	tab[n++]=sprintf(nil "FIELD path_list")
	tab[n++]=sprintf(nil "FLOC 1 3")
	tab[n++]=sprintf(nil "LIST %L 30 10" "")
	tab[n++]=sprintf(nil "OPTIONS MULTISELECT | SORT")
	tab[n++]=sprintf(nil "ENDFIELD")
	tab[n++]=sprintf(nil "")
	tab[n++]=sprintf(nil "FIELD btn_add")
	tab[n++]=sprintf(nil "FLOC 33 8")
	tab[n++]=sprintf(nil "MENUBUTTON %L 15 3" "ADD Path")
	tab[n++]=sprintf(nil "ENDFIELD")
	tab[n++]=sprintf(nil "FIELD btn_del")
	tab[n++]=sprintf(nil "FLOC 33 12")
	tab[n++]=sprintf(nil "MENUBUTTON %L 15 3" "Delete Path")
	tab[n++]=sprintf(nil "ENDFIELD")
	tab[n++]=sprintf(nil "TEXT")
	tab[n++]=sprintf(nil "TLOC 32 3")
	tab[n++]=sprintf(nil "FSIZE 15 5")
	tab[n++]=sprintf(nil "INFO process_info 15")
	tab[n++]=sprintf(nil "ENDTEXT")
	tab[n++]=sprintf(nil "FIELD btn_ok")
	tab[n++]=sprintf(nil "FLOC 5 25")
	tab[n++]=sprintf(nil "MENUBUTTON %L 15 3" "Run")
	tab[n++]=sprintf(nil "ENDFIELD")
	tab[n++]=sprintf(nil "FIELD btn_cancel")
	tab[n++]=sprintf(nil "FLOC 25 25")
	tab[n++]=sprintf(nil "MENUBUTTON %L 15 3" "Cancel")
	tab[n++]=sprintf(nil "ENDFIELD")
	tab[n++]=sprintf(nil "ENDTILE")
	tab[n++]=sprintf(nil "ENDFORM")
	tab[n++]=sprintf(nil "")
	)
	vec=makeVector(length(tab)) foreach(a tab vec[a]=tab[a])
	ls=vectorToList(vec) tab=vec=nil
	filename=sprintf(nil "%s/%s_temp.form" axlOSSlash(getTempDir()) getStrCurrentTime())
	op=outfile(filename)
	fprintf(op "%s" buildString(ls "\n"))
	close(op)
	filename
))

; 2. Form Callback 
procedure( HCIS_ErrPadSym_CallBack(f)
let((path v dir) prog(()
	case(f->curField
		(("" "btn_cancel") axlFormClose(f) return(nil))
		("btn_ok"
			jeff_padstack_drill_tol_check_Run()
			axlFormClose(f) return(nil)
		)
		("btn_add"
			dir=getWorkingDir()
			path=axlDMDirectoryBrowse(glb_startDir t ?title "Pick a directory")
			changeWorkingDir(dir)
			when(path axlFormListAddItem(f "path_list" path -1) glb_startDir=path)
			glb_checkPaths=oxlFormListGetAllItems(f "path_list")
		)
		("btn_del"
			ls=axlFormListGetSelItems(f "path_list")
			axlFormListDeleteItem(f "path_list" ls)
			glb_checkPaths=oxlFormListGetAllItems(f "path_list")
		)
	)
	v=if(axlFormListGetSelItems(f "path_list") t nil)
	axlFormSetFieldEditable(Form "btn_del" v)
	axlFormSetFieldEditable(Form "btn_ok" if(glb_checkPaths t nil))
)))

; 3. 檢查功能 
procedure( jeff_padstack_drill_tol_check_Run()
prog(( dir extpath viewfile batfile excelFile, pathobj tab o tabErrPads tabErrSyms sheets ) 
	
	Henry_CreatePadTxts = nil ;t 
	Henry_DeletePadTxts = nil 
	Henry_CreateSymTxts = nil ;t 
	Henry_DeleteSymTxts = nil 
	
	dir=getWorkingDir()
	extpath=sprintf(nil "%s/extracta" dir)
	viewfile=sprintf(nil "%s/pad_view.txt" dir) 
	batfile=sprintf(nil "%s/run.bat" dir)
	excelFile=sprintf(nil "%s/pad_tol_check_%s.xlsx" dir getStrCurrentTime())
	
	;===== 3-1. 全部的檢查地址和鏡像地址 
	pathobj = HCIS_ErrPadSym_DirsMirDirs( glb_checkPaths extpath ) 
	
	;===== 3-2. 全部的Pads文件和PadTxts文件 
	tab = HCIS_ErrPadSym_PadsPadTxtsFullPath( pathobj ) 
	
when( Henry_CreatePadTxts 
	;===== 刪除鏡像路徑下全部的PadTxts舊文件 (只刪PadTxts文件,資料夾保留,不會刪SymTxts) 
	foreach(o pathobj deleteDirs(o->txtpath) oxlCreateDir(o->txtpath))				;測試--臨時注銷 
	;===== 3-3. 生成"pad_view.txt"標記檔, 生成並執行"run.bat"==>生成PadTxts文件 
	HCIS_ErrPadSym_CrtPadTxtsByBat( viewfile batfile tab )							;測試--臨時注銷 
) 
	
	;===== 3-4. 檢查錯誤的Pads 
	tabErrPads = HCIS_ErrPadSym_ChkErrPads( tab extpath ) 
	when( length(tabErrPads)==0  axlMsgPut("No Error.") return(nil)	)
	;_Hprintf2( tableToList(tabErrPads) 1 t sprintf(nil "%s/Henry_ErrPads.txt" dir) )  ;共計57個 
	
	;===== 刪除全部的PadTxts文件, 刪除"pad_view.txt"和"run.bat" 
when( Henry_DeletePadTxts 
	deleteDirs(extpath)															;測試--臨時注銷
	foreach(a list(batfile viewfile) deleteFile(a))								;測試--臨時注銷
) 
	
	;===== 創建extracta資料夾 
	oxlCreateDir(extpath)
	;===== 3-5. 生成並執行"run.bat"==>生成SymTxts文件 
when( Henry_CreateSymTxts 
	HCIS_ErrPadSym_CrtSymTxtsByBat( tabErrPads batfile )							;測試--臨時注銷
) 
	
	;===== 3-6. 檢查錯誤的零件 
	tabErrSyms = HCIS_ErrPadSym_ChkErrSyms( tabErrPads ) 
	
	;===== 3-7. 轉換為Excel所需格式 
	sheets = HCIS_ErrPadSym_ChgToExcelFmt( tabErrSyms ) 
	
	;=== 刪除SymTxts, 刪除"run.bat, 刪除extracta資料夾 
when( Henry_DeleteSymTxts 
	ns=sort(tabErrPads->? 'lessp), foreach(i ns  o=tabErrPads[i], txtFile=o->tmpfile, when(isFile(txtFile) deleteFile(txtFile))) 
	foreach(a list(batfile) deleteFile(a)) 
	deleteDirs(extpath) 
) 
	
	;===== 生成Excel Report 
	allegroExcel( "w" excelFile sheets)
	axlMsgPut("Finish - Output Error Report (%L)" excelFile)
	axlShell(sprintf(nil "http %L" excelFile))
	
	return(t) 
)) 

;===== 3-1. 全部的檢查地址和鏡像地址  
procedure( HCIS_ErrPadSym_DirsMirDirs( checkPaths extpath ) 
prog(( pathobj path path1 txtpath ) 
	pathobj=nil
	rexCompile("[:]")
	foreach(path checkPaths
		path1=rexReplace(path "" 0)
		when(substring(path1 1 2)=="//" path1=substring(path1 3))
		txtpath=sprintf(nil "%s/%s" extpath path1)
		pathobj=append1(pathobj list(nil 'path path 'txtpath txtpath))
	)
	return( pathobj ) 
)) 

;===== 3-2. 全部的Pads文件和PadTxts文件 
procedure( HCIS_ErrPadSym_PadsPadTxtsFullPath( pathobj ) 
prog(( tab n o path txtpath a fs file padname txtname ) 
	tab=makeTable("" nil) 
	n=0
	foreach(o pathobj
		path=o->path 
		txtpath=o->txtpath
		fs=setof(a getDirFiles(path) substring(lowerCase(a) -4)==".pad")
		foreach(file fs
			padname=sprintf(nil "%s/%s" path file)
			txtname=sprintf(nil "%s/%s.txt" txtpath file)
			tab[n++]=list(nil 'padname padname 'txtname txtname)
		)
	) 
	return( tab ) 
)) 

;===== 3-3. 生成"pad_view.txt"標記檔, 生成並執行"run.bat"==>生成PadTxts文件 
procedure( HCIS_ErrPadSym_CrtPadTxtsByBat( viewfile batfile tab ) 
prog(( op exefile ns count i o ) 
	op=outfile(viewfile) 
	fprintf(op "PAD_DEF\n") 
	close(op)
	;; 
	exefile=sprintf(nil "%s/extracta.exe" axlOSSlash(AllegroBinDir))
	op=outfile(batfile)
	fprintf(op "set cdsroot=%s\n"  axlGetVariable("cdsroot"))
	fprintf(op "set path=%s\n"  axlGetVariable("path"))
	ns=sort(tab->? 'lessp) 
	count=length(ns)
	foreach(i ns 
		o=tab[i] 
		fprintf(op "REM %d / %d\n" i count ) 
		fprintf(op "%s %s %s %s\n" exefile o->padname viewfile o->txtname)
	)
	close(op)
	shell(batfile)
	return(t) 
)) 

;===== 3-4. 檢查錯誤的Pads 
procedure( HCIS_ErrPadSym_ChkErrPads( tab extpath ) 
prog(( tabErrPads en i o txtfile atab a j p dstype ds_mil ) 
	tabErrPads=makeTable("" nil) 
	en=0 
	foreach(i tab 
		o=tab[i] 
		txtfile=o->txtname
		;===== 3-4-1. 讀取單個PadTxt數據 
		atab=jeff_padstack_drill_tol_check_txt2tabdata(txtfile)
		j=car(setof(a atab atab[a]->LAYER=="~DRILL"))
		if( j then 
			p=atab[j]
			dstype=get(p "APADSHAPE1")	; NONE, XXXXX 
			ds_mil=axlMKS2UU(sprintf(nil "%s %s" get(p "PADSHAPE1") get(p "UNIT") ))
			when( dstype!="NONE" && ds_mil==0.0 
				tabErrPads[en++]=list(nil 'FILENAME get(p "FILENAME") 'drilltype dstype 'drillsize ds_mil 
										'tmpfile sprintf(nil "%s/%d.txt" extpath en)
							)
			)
		else 
			axlMsgPut("##########  %L, %L" i txtfile) 
		) 
	)
	return( tabErrPads ) 
)) 
;===== 3-4-1. 讀取單個PadTxt數據 
procedure( jeff_padstack_drill_tol_check_txt2tabdata(txtfile)
prog((ip tab keys n S s line key v i a ) 
	ip=infile(txtfile) 
	when(!ip return(nil))
	;; 
	tab=makeTable("" nil) 
	n=0
	keys=nil
	rexCompile("[!]")
	while(gets(S ip)
		s=rexReplace(S "," 0)
		line=axlStringCSVParse(s t)
		case(car(line)
			("A" 	keys=append(line list("FILENAME" "UNIT" "DEG")))
			("J" 	filename=nth(1 line) 
					unit=nth(8 line) 
					deg=nth(11 line)
			)
			("S"	o=list(nil)
					for(i 0 length(keys)-1
						key=nth(i keys)
						v=nth(i line)
						putprop(o v key)
					)
					putprop(o filename "FILENAME")
					putprop(o unit "UNIT")
					putprop(o deg "DEG")
					tab[n++]=o
			)
		)
	)
	close(ip)
	return(tab) 
))

;===== 3-5. 生成並執行"run.bat"==>生成SymTxts文件 
procedure( HCIS_ErrPadSym_CrtSymTxtsByBat( tabErrPads batfile ) 
prog(( conStr dbsearchExe txtFile op ns count i o filename l1 padname searchStr runstr ) 
	conStr=sprintf(nil "Provider=sqloledb;Server=ptw-cae-01;Database=CIS_Server_FileInfo;Uid=CADCAM;Pwd=dfm#cae20135;" )
	dbsearchExe=sprintf(nil "%s/other_apps/SQLDbQueryTool.exe" axlOSSlash(PcbenvDir))
	op=outfile(batfile) 
	ns=sort(tabErrPads->? 'lessp) 
	count=length(ns)
	foreach(i ns 
		o=tabErrPads[i]
		filename=o->FILENAME 
		txtFile=o->tmpfile
		;; 
		l1=axlDMFileParts(filename)
		padname=upperCase(nth(1 l1)) 
		;searchStr=sprintf(nil "SELECT a.SymbolName FROM CISLib_Dra a left join CISLib_Dra_Pin b on a.CompID=b.CompID where b.Padstack='%s' " padname)
		searchStr=sprintf(nil "SELECT a.SymbolName, c.FilePath, b.Padstack FROM CISLib_Dra a LEFT JOIN CISLib_Dra_Pin b ON a.CompID = b.CompID LEFT JOIN CISLib_File c ON a.FileID = c.FileID WHERE b.Padstack = '%s'" padname)		
		runstr=sprintf(nil "%s %L %L %L" dbsearchExe conStr searchStr txtFile)
		fprintf(op "REM %d / %d\n" i count ) 
		fprintf(op "%s\n" runstr)
	)
	close(op)
	shell(batfile)
	return(t) 
)) 

;===== 3-6. 檢查錯誤的零件 
procedure( HCIS_ErrPadSym_ChkErrSyms( tabErrPads ) 
prog(( ns tabErrSyms n i o filename PthIFs padname path txtFile SymIFLst OneSymIF DraFile SymPth SymNm ) 
	tabErrSyms=makeTable("" nil)
	n=0
	ns=sort(tabErrPads->? 'lessp)
	foreach(i ns 
		o=tabErrPads[i] 
		filename = o->FILENAME 
		dstype   = o->drilltype 
		ds_mil   = o->drillsize 
		txtFile  = o->tmpfile 
		;; 
		PthIFs = axlDMFileParts(filename)
		padPth = nthelem(1 PthIFs) 
		padNm  = upperCase(nthelem(2 PthIFs)) 
		;===== 統計單份SymTxt中全部的SymNms 
		SymIFLst = HCIS_ErrPadSym_GetAllSyms( txtFile ) 
		if( SymIFLst then 
			foreach( OneSymIF SymIFLst
				DraFile = nthelem(2 OneSymIF) 
				;; 
				PthIFs = axlDMFileParts(DraFile) 
				SymPth = nthelem(1 PthIFs) 
				SymNm  = upperCase(nthelem(2 PthIFs)) 
				tabErrSyms[n++]=list(nil 'pad_path padPth 'pad_name padNm 'drilltype dstype 'drillsize ds_mil 
											'sym_path SymPth 'sym_name SymNm ) 
			)
		else 
				tabErrSyms[n++]=list(nil 'pad_path padPth 'pad_name padNm 'drilltype dstype 'drillsize ds_mil 
											'sym_path "" 'sym_name "" ) 
		) 
	)
	return( tabErrSyms ) 
)) 
;===== 3-6-1. 統計單份SymTxt中全部的SymNms 
procedure( HCIS_ErrPadSym_GetAllSyms( txtFile ) 
prog(( SymIFLst ip Ln Line LineLst SymNm DraFile PadName ) 
	SymIFLst=nil
	when( isFile(txtFile) 
		ip=infile(txtFile)
		Ln = 0 
		while( gets(Line ip) 
			Ln++ 
			when( Ln>1 
				LineLst = parseString(Line ",\n") 
				SymNm   = nthelem(1 LineLst) 
				DraFile = nthelem(2 LineLst)  ;.dra文件完整路徑 
				PadName = nthelem(3 LineLst) 
				;; 
				SymNm   = upperCase(or(SymNm "")) 
				DraFile = or(DraFile "") 
				PadName = or(PadName "") 
				SymIFLst = cons(list(SymNm DraFile PadName) SymIFLst) 
			) 
		)
		close(ip) 
	)
	SymIFLst=unique(reverse(SymIFLst)) 
	return( SymIFLst ) 
)) 

;===== 3-7. 轉換為Excel所需格式 
/*	
	Report格式參考:  D:\pro.172\g1-m16p_20_20260729._m15\symbolNames_00.txt 
	ns=setof(a tab tab[a]->type=="normal")	;篩選並返回序號 
	ns=sort(tab->? 'lessp)					;序號排序 
	foreach(a tab  o=tab[a])					;循環序號 
*/ 
procedure( HCIS_ErrPadSym_ChgToExcelFmt( tabErrSyms ) 
prog(( sheets tab1 n a ns i o vec data sheet1 ) 
	tab1=makeTable("" nil) 
	n=0
	tab1[n++]=list( "PAD Path"  "PAD Name"  "Drill Type"  "Drill Size"  "Symbol Path"  "Symbol Name" ) 
	ns=sort(tabErrSyms->? 'lessp)
	foreach(i ns 
		o=tabErrSyms[i] 
		tab1[n++]=list(o->pad_path o->pad_name o->drilltype o->drillsize o->sym_path o->sym_name) 
	) 
	vec=makeVector(length(tab1)) 
	foreach(a tab1 vec[a]=tab1[a]) 
	data=vectorToList(vec) 
	sheet1=list(nil 'SheetName "Sheet" 'data data)
	;; 
	tab1=vec=nil
	;; 
	sheets=nil
	sheets=append1(sheets sheet1)
	return( sheets ) 
)) 






















