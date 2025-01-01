# Custom-button-in-the-Quote-module-triggers-the-process.
Automate the process of combining mail merge with doc file and send it to ZOHO sign for customer Signature by click custom button in Zoho CRM 
string button.send_for_signature(String qid)
{
try 
{
	quotes = invokeurl
	[
		url :"https://crm.zoho.com.au/crm/v6/Quotes/" + qid
		type :GET
		connection:"zohocrm"
	];
	get_quotes_data = quotes.get("data");
	subtotal = quotes.getJSON("data").getJSON("Sub_Total");
	if(get_quotes_data != null && get_quotes_data != "")
	{
		plans_Doc = get_quotes_data.getJSON("Plans_Doc");
		if(plans_Doc == null || plans_Doc.isEmpty())
		{
			return "Please upload the Plans Doc";
		}
		fileList = List();
		file_id = plans_Doc.getJSON("File_Id__s");
		//******************************Get Doc File******************************************//
		response_attach = invokeurl
		[
			url :"https://www.zohoapis.com.au/crm/v3/files?id=" + file_id
			type :GET
			connection:"zohocrm"
		];
		// 		info response_attach;
		//******************************Map for Merger Template*************************//
		contact_deatails = ifnull(get_quotes_data.getJSON("Contact_Name"),"");
		if(contact_deatails != null)
		{
			contact_name = contact_deatails.getJSON("name");
			contact_id = contact_deatails.getJSON("id");
		}
		contact = zoho.crm.getRecordById("Contacts",contact_id);
		contact_email = contact.get("Email");
		// contact_email = "amanucs4@gmail.com";
		fields = Map();
		fields.put("Contact_Name.Full_Name",contact_name);
		fields.put("Owner",ifnull(get_quotes_data.getJSOn("Owner").getJSON("name"),""));
		fields.put("Subject",ifnull(get_quotes_data.getJSOn("Subject"),""));
		fields.put("Contact_Name.Mobile",ifnull(contact.get("Mobile"),"0"));
		fields.put("Contact_Name.Mailing_Street",ifnull(contact.get("Mailing_Street"),""));
		fields.put("Contact_Name.Mailing_City",ifnull(contact.get("Mailing_City"),""));
		fields.put("Contact_Name.Mailing_State",ifnull(contact.get("Mailing_State"),""));
		fields.put("Contact_Name.Mailing_Zip",ifnull(contact.get("Mailing_Zip"),""));
		fields.put("id",qid);
		//*************************** Add product details*************************//
		quoted_Items = get_quotes_data.getJSON("Quoted_Items");
		subformList = List();
		for each  item in quoted_Items
		{
			item_map = Map();
			item_map.put("Quoted_Items.Room_Name",item.get("Room_Name"));
			item_map.put("Quoted_Items.Product_Name",item.getJSON("Product_Name").get("name"));
			item_map.put("Quoted_Items.Height",ifnull(item.get("Height"),0));
			item_map.put("Quoted_Items.Depth",ifnull(item.get("Depth"),0));
			item_map.put("Quoted_Items.Comments1",ifnull(item.get("Comments1"),""));
			item_map.put("Quoted_Items.Net_Total",ifnull(item.get("Net_Total"),0));
			subformList.add(item_map);
		}
		fields.put("Sub_Total",subtotal);
		fields.put("Discount",get_quotes_data.getJSON("Discount"));
		fields.put("GST",ifnull(get_quotes_data.getJSON("GST1"),0.0));
		fields.put("Grand_Total",ifnull(get_quotes_data.getJSON("Grand_Total"),0.0));
		fields.put("Quoted_Items",subformList);
		//************************ Prepare merge data***********************//
		data_map = Map();
		data_map.put("data",fields);
		mailData = Map();
		mailData.put("subject","Document");
		mailData.put("merge_data",data_map);
		mergeAndDownload = zoho.writer.mergeAndDownload("ep9fgd3ff19f4f55141b98034bdb0e642f7b4","pdf",mailData,"zoho_writer");
		// 		info "mergeAndDownload" + mergeAndDownload;
		//************************ Prepare combine***********************//
		mergeAndDownload.setParamName("files");
		response_attach.setParamName("files");
		output_settings = Map();
		output_settings.put("name","combined-document");
		input_options = Map();
		document_1 = Map();
		document_1.put("page_ranges","1,2");
		document_2 = Map();
		document_2.put("page_ranges","1,2");
		input_options.put("1",document_1);
		input_options.put("2",document_2);
		paramList = list();
		paramList.add({"paramName":"files1","content":mergeAndDownload});
		paramList.add({"paramName":"files","content":response_attach});
		paramList.add({"paramName":"output_settings","content":output_settings.toString(),"Content-Type":"application/json","stringPart":"true"});
		paramList.add({"paramName":"input_options","content":input_options.toString(),"Content-Type":"application/json","stringPart":"true"});
		combinepdf_response = invokeurl
		[
			url :"https://www.zohoapis.com.au/writer/api/v1/documents/pdf/combine"
			type :POST
			files:paramList
			connection:"zoho_writer"
		];
		if(combinepdf_response != null)
		{
			status_check_url = combinepdf_response.get("status_check_url");
			// 			info status_check_url;
			list12 = {1,2,3,4,5};
			for each  resc in list12
			{
				getUrl("https://httpstat.us/200?sleep=5000");
				ttt = invokeurl
				[
					url :status_check_url
					type :GET
					connection:"zoho_writer"
				];
				download_link = ttt.get("download_link");
			}
			res_down = invokeurl
			[
				url :download_link
				type :GET
				connection:"zoho_writer"
			];
		}
		else
		{
			return "Upload Plan Doc in PDF !";
		}
		//******************************Send For Zoho Sign***********************//
		actionlist1 = List();
		reciMap = Map();
		reciMap.put("recipient_name","Recipient Name");
		reciMap.put("recipient_email",contact_email);
		reciMap.put("action_type","SIGN");
		reciMap.put("signing_order",0);
		actionlist1.add(reciMap);
		q = List();
		q.add(res_down);
		data_Map1 = Map();
		requests = Map();
		requests.put("request_name","WD Quote_NEW.pdf");
		requests.put("expiration_days",30);
		requests.put("is_sequential",true);
		requests.put("email_reminders",true);
		requests.put("reminder_period",8);
		requests.put("actions",actionlist1);
		datamap1 = Map();
		datamap1.put("requests",requests);
		requestMap1 = Map();
		requestMap1.put("data",datamap1);
		res = zoho.sign.createDocument(q,requestMap1,"zoho_writer");
		req_id = res.get('requests').get('request_id');
		// 		info "req_id" + req_id;
		doc_id = res.get("requests").get("document_ids").get(0).get("document_id");
		action_id_lead = res.get("requests").get("actions").get(0).get("action_id").toLong();
		//***************************static data for testing*********************************
		// 		document = zoho.sign.getDocumentById(12930000000108145);
		// 		info document;
		// 		doc_id = document.get("requests").get("document_ids").get(0).get("document_id");
		// 		req_id = document.get('requests').get('request_id');
		// 		action_id_lead = document.get("requests").get("actions").get(0).get("action_id").toLong();
		//*******************Create Zoho Sing Documents**************************//
		download_document = zoho.sign.downloadDocument(req_id);
		zoho_sign_name = res.get('requests').get('request_name');
		if(!get_quotes_data.getJSON("Account_Name").isNull())
		{
			acc_id = get_quotes_data.getJSON("Account_Name").get("id");
			if(acc_id.isNull())
			{
				acc_id = "";
			}
		}
		if(!get_quotes_data.getJSON("Deal_Name").isNull())
		{
			deal_id = get_quotes_data.getJSON("Deal_Name").get("id");
			if(deal_id.isNull())
			{
				deal_id = "";
			}
		}
		recordmap = Map();
		recordmap.put("Name",zoho_sign_name);
		recordmap.put("zohosign__Account",acc_id);
		recordmap.put("zohosign__Quote",qid);
		recordmap.put("zohosign__Deal",deal_id);
		recordmap.put("zohosign__Owner",get_quotes_data.getJSON("Owner").get("id"));
		recordmap.put("zohosign__Date_Sent",zoho.currentdate.toString("yyyy-MM-dd"));
		recordmap.put("zohosign__ZohoSign_Document_ID",doc_id);
		recordmap.put("zohosign__Deal",deal_id.toString());
		recordmap.put("zohosign__Document_Deadline",today.addDay(30));
		recordmap.put("zohosign__Document_Status","Out for Signature");
		recordmap.put("zohosign__Module_Name","Quotes");
		recordmap.put("zohosign__Document_Note","Please find the document to be signed here");
		recordmap.put("Check_Date",today.addDay(1));
		recordmap.put("Frequency",0);
		recordmap.put("Request_ID",req_id);
		//********* Create a Record in the "ZohoSign Documents" module in Zoho CRM***************//
		create_sign = zoho.crm.createRecord("zohosign__ZohoSign_Documents",recordmap,{"trigger":{"workflow"}});
		// 		info create_sign;
		sing_id = create_sign.get("id");
		download_document.setParamName("file");
		update_sing_doc = zoho.crm.attachFile("zohosign__ZohoSign_Documents",sing_id,download_document);
		// 		info  update_sing_doc;
		update_sing_quotes = zoho.crm.updateRecord("zohosign__ZohoSign_Documents",sing_id,{"zohosign__Quote":qid},{"trigger":{"workflow"}});
		// 		info update_sing_quotes;
		//*******************Sending For Signature**************************//
		fields_list = List();
		//----------------------//
		field_1 = Map();
		field_1.put("x_coord",227);
		field_1.put("field_type_id","12930000000000279");
		field_1.put("abs_height",16);
		field_1.put("field_category","image");
		field_1.put("field_label","Signature");
		field_1.put("field_name","Signature");
		field_1.put("is_mandatory",true);
		field_1.put("page_no",1);
		field_1.put("document_id",doc_id);
		field_1.put("is_draggable",false);
		field_1.put("field_name","");
		field_1.put("y_value",62.310608);
		field_1.put("abs_width",180);
		field_1.put("action_id",action_id_lead);
		field_1.put("width",24.509804);
		field_1.put("y_coord",430);
		field_1.put("field_type_name","Signature");
		field_1.put("x_value",37.132351);
		field_1.put("is_resizable",true);
		field_1.put("height",2.083333);
		fields_list.add(field_1);
		//*******************Signature**************************//
		field_2 = Map();
		field_2.put("x_coord",221);
		field_2.put("field_type_id","12930000000000279");
		field_2.put("abs_height",16);
		field_2.put("field_category","image");
		field_2.put("field_name","Signature");
		field_2.put("field_label","Signature");
		field_2.put("is_mandatory",true);
		field_2.put("page_no",1);
		field_2.put("document_id",doc_id);
		field_2.put("y_value",66.28788);
		field_2.put("abs_width",180);
		field_2.put("action_id",action_id_lead);
		field_2.put("width",24.509804);
		field_2.put("y_coord",450);
		field_2.put("field_type_name","Signature");
		field_2.put("x_value",36.151962);
		field_2.put("is_resizable",true);
		field_2.put("height",2.083333);
		fields_list.add(field_2);
		//*******************Full name**************************//
		field_3 = Map();
		field_3.put("x_coord",98);
		field_3.put("field_type_id","12930000000000291");
		field_3.put("abs_height",16);
		field_3.put("is_mandatory",true);
		field_3.put("page_no",1);
		field_3.put("document_id",doc_id);
		field_3.put("field_name","Full name");
		field_3.put("y_value",70.265152);
		field_3.put("abs_width",150);
		field_3.put("action_id",action_id_lead);
		field_3.put("width",24.509804);
		field_3.put("y_coord",490);
		field_3.put("field_type_name","Name");
		field_3.put("x_value",16.053921);
		field_3.put("height",2.083333);
		fields_list.add(field_3);
		//*******************Date**************************//
		field_4 = Map();
		field_4.put("action_id",action_id_lead);
		field_4.put("field_name","Sign Date");
		field_4.put("field_label","Sign date");
		field_4.put("field_type_name","Date");
		field_4.put("document_id",doc_id);
		field_4.put("is_mandatory",true);
		field_4.put("height",2.083333);
		field_4.put("x_coord",96);
		field_4.put("y_coord",500);
		field_4.put("abs_width",150);
		field_4.put("abs_height",16);
		field_4.put("x_value",15.686275);
		field_4.put("y_value",74.242424);
		field_4.put("page_no",1);
		// 		field_4.put("default_value",zoho.currentdate);
		fields_list.add(field_4);
		fild_map = Map();
		action_map = Map();
		fild_map.put("action_type","SIGN");
		fild_map.put("recipient_name",contact_name);
		fild_map.put("private_notes","Please find the document to be signed here");
		fild_map.put("cloud_provider_name","Zoho Sign");
		fild_map.put("recipient_email",contact_email);
		fild_map.put("send_completed_document",true);
		fild_map.put("action_id",action_id_lead);
		fild_map.put("signing_order",0);
		fild_map.put("fields",fields_list);
		action_list = List();
		action_list.add(fild_map);
		action_map.put("request_name","WD Quote_NEW.pdf");
		action_map.put("actions",action_list);
		request_map = Map();
		request_map.put("requests",action_map);
		dataMap = Map();
		dataMap.put("data",request_map);
		response_sign = zoho.sign.submitRequest(req_id.toLong(),dataMap);
		if(response_sign.getJSON("requests").get("request_status") == "inprogress")
		{
			return "Quote and Plans Sent for Customer Signature";
		}
		//***********************Create record in  Sign Documents module*************************//
	}
}
catch (e)
{
	info "catch" + e;
	return "Error occurred: " + e.toString();
}
return "";
}
