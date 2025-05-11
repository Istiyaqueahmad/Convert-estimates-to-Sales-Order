# Convert-estimates-to-Sales-Order

// Zoho provided script to convert Estimate into Sales Order

estimateID = estimate.get("estimate_id");

estimatedate = estimate.get("date").toDate();

organizationID = organization.get("organization_id");

customerID = estimate.get("customer_id");

estimateDetails = invokeurl
[
	url :"https://books.zoho.com/api/v3/estimates/" + estimateID + "?organization_id=" + organizationID
	type :GET
	connection:"zohobooks"
];

estimateDetails = estimateDetails.get("estimate");
dis = estimateDetails.get("is_discount_before_tax");
info estimateDetails;
cpList = List();
cpList = estimateDetails.get("contact_persons").toList();
estimate_number = estimateDetails.get("estimate_number");
discountType = estimateDetails.get("discount_type");
discount = estimateDetails.get("discount");
salespersonID = estimateDetails.get("salesperson_id");
lineItems = estimateDetails.get("line_items").toList();
lineList = List();
for each  find in lineItems
{
	lineMap = Map();
	itemCustomFields = find.get("item_custom_fields").toList();
	itemCF = List();
	for each  findCF in itemCustomFields
	{
		itemLab = findCF.get("label");
		itemVal = findCF.get("value");
		iMap = Map();
		iMap.put("label",itemLab);
		iMap.put("value",itemVal);
		itemCF.add(iMap);
	}
	find.remove("item_custom_fields");
	find.remove("line_item_id");
	find.put("item_custom_fields",itemCF);
	lineList.add(find);
}
getSOCustomFields = invokeurl
[
	url :"https://books.zoho.com/api/v3/settings/fields?entity=salesorder&organization_id=" + organizationID
	type :GET
	connection:"zohobooks"
];
// info getSOCustomFields;
getSOCustomFields = getSOCustomFields.get("fields").toList();
estimateCustomFields = estimateDetails.get("custom_fields").toList();
cfList = List();
for each  findEstimateCF in estimateCustomFields
{
	estimateCF_isActive = false;
	estimateCF_dataType = 0;
	estimateCF_label = 0;
	estimateCF_value = "";
	estimateCF_isActive = findEstimateCF.get("is_active");
	estimateCF_dataType = findEstimateCF.get("data_type");
	estimateCF_label = findEstimateCF.get("label");
	estimateCF_value = findEstimateCF.get("value");
	for each  findSOCF in getSOCustomFields
	{
		SOCF_isActive = false;
		SOCF_dataType = 0;
		SOCF_label = 0;
		SOCF_value = "";
		SOCF_isActive = findSOCF.get("is_active");
		SOCF_dataType = findSOCF.get("data_type");
		SOCF_label = findSOCF.get("field_name_formatted");
		if(estimateCF_isActive = true && SOCF_isActive == true && estimateCF_dataType == SOCF_dataType && estimateCF_label == SOCF_label && !estimateCF_value.isNull())
		{
			cMap = Map();
			cMap.put("label",SOCF_label);
			cMap.put("value",estimateCF_value);
			cfList.add(cMap);
		}
	}
}
json = Map();
json.put("customer_id",customerID);
json.put("estimate_id",estimateID);
if(estimate.get("zcrm_potential_id"))
{
	json.put("zcrm_potential_id",estimate.get("zcrm_potential_id"));
}
if(discountType == "entity_level")
{
	json.put("discount",discount);
	json.put("is_discount_before_tax",dis);
	json.put("discount_type","entity_level");
}
else if(discountType == "item_level")
{
	json.put("is_discount_before_tax",dis);
	json.put("discount_type","item_level");
}
json.put("reference_number",estimate_number);
json.put("line_items",lineList);
if(!salespersonID.isNull())
{
	json.put("salesperson_id",salespersonID);
}
if(!cpList.isEmpty())
{
	json.put("contact_persons",cpList);
}
if(!cfList.isEmpty())
{
	json.put("custom_fields",cfList);
}
params = Map();
params.put("JSONString",json);
//info params;
createSO = invokeurl
[
	url :"https://books.zoho.com/api/v3/salesorders?organization_id=" + organizationID
	type :POST
	parameters:params
	connection:"zohobooks"
];
info createSO.get("message");
soCode = createSO.get("code");
// Error occurred if soCode is not 0, Sales Order wasn't created
if(soCode != 0)
{
	cmt = Map();
	cmt.put("description","Failed to create a salesorder");
	cmtMap = Map();
	cmtMap.put("JSONString",cmt);
	comment = invokeurl
	[
		url :"https://books.zoho.com/api/v3/estimates/" + estimateID + "/comments?organization_id=" + organizationID
		type :POST
		parameters:cmtMap
		connection:"zohobooks"
	];
	info "cmt==" + comment;
	// Sales Order was created	
}
else
{
	// Get Sales Order information, then create any CRM Treadmill Records as necessary
	salesorder = createSO.get("salesorder");
	info salesorder;
	salesorderdate = salesorder.get("date").toDate();
	loop = {1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46,47,48,49,50};
	// info salesorder;
	customerID = salesorder.get("customer_id");
	customer = zoho.books.getRecordsByID("Contacts",organizationID,customerID,"zohobooks");
	// info customer;
	accountID = customer.get("contact").get("zcrm_account_id");
	dealID = 0;
	if(!salesorder.get("zcrm_potential_id").isNull())
	{
		dealID = salesorder.get("zcrm_potential_id");
	}
	info "deal: " + dealID;
	contactID = 0;
	if(dealID != 0)
	{
		deal = zoho.crm.getRecordById("Deals",dealID);
		if(!deal.get("Contact_Name").isNull())
		{
			contactID = deal.get("Contact_Name").get("id");
		}
		info contactID;
	}
	// Create a map containing all Account details necessary for creating a Treadmill Record in CRM
	crmRecord = Map();
	crmRecord.put("Account_Name1",accountID);
	// 		info "Contact: " + contactID;
	if(contactID != 0)
	{
		crmRecord.put("Contacts",contactID);
	}
	crmRecord.put("Purchase_Date",salesorder.get("date"));
	billingAddress = salesorder.get("billing_address");
	// 		info "Billing Address: "+billingAddress;
	crmRecord.put("Billing_Street",billingAddress.get("address"));
	crmRecord.put("Billing_City",billingAddress.get("city"));
	crmRecord.put("Billing_State",billingAddress.get("state"));
	crmRecord.put("Billing_Code",billingAddress.get("zip"));
	crmRecord.put("Billing_Country",billingAddress.get("country"));
	shippingAddress = salesorder.get("shipping_address");
	crmRecord.put("Shipping_Street",shippingAddress.get("address"));
	crmRecord.put("Shipping_City",shippingAddress.get("city"));
	crmRecord.put("Shipping_State",shippingAddress.get("state"));
	crmRecord.put("Shipping_Code",shippingAddress.get("zip"));
	crmRecord.put("Shipping_Country",shippingAddress.get("country"));
	// Get line item data
	items = salesorder.get("line_items");
	info "Line Items: " + items;
	newTreadmills = List();
	for each  item in items
	{
		// 		serialNumbers = item.get("serial_numbers");
		// Category isn't stored on the line item, retreive Item data
		productID = item.get("item_id");
		product = zoho.books.getRecordsByID("Items",organizationID,productID,"zohobooks");
		// 		info product;
		category = product.get("item").get("category_name");
		info "Product Category: " + category;
		if(category == "Full Treadmills")
		{
			info "Treadmill found";
			// 	Start with base CRM record
			itemCopy = crmRecord;
			// Add all line item data to base CRM record
			// 			itemCopy.putAll(item);
			itemCopy.put("Name",item.get("name"));
			quantity = item.get("quantity");
			// Add all quantity of items to New Treadmill list with help of Loop list
			for each  iteration in loop
			{
				if(iteration <= quantity)
				{
					newTreadmills.add(itemCopy);
				}
			}
		}
		else
		{
			info "Not a treadmill";
		}
	}
	// info serialItems;
	treadmillIDs = list();
	for each  newTreadmill in newTreadmills
	{
		info "Creating Treadmill";
		cresponse = zoho.crm.createRecord("Installed_Base",newTreadmill,{"trigger":{"workflow"}},"zohocrm");
		info cresponse;
		treadmillIDs.add(cresponse.get("id"));
	}
	if(dealID != 0)
	{
		for each  tm in treadmillIDs
		{
			// Associate Deal with Treadmill Record
			multimap = Map();
			multimap.put("Serial_Numbers",tm);
			multimap.put("Deals",dealID);
			xDealRecord = zoho.crm.createRecord("Deals_X_Treadmill_Records",multimap);
			info xDealRecord;
		}
	}
	if(contactID != 0)
	{
		for each  tm in treadmillIDs
		{
			contactMap = Map();
			contactMap.put("Treadmills",tm);
			contactMap.put("All_Contacts",contactID);
			xrecord = zoho.crm.createRecord("Treadmill_X_Contacts",contactMap);
			info xrecord;
		}
	}
}
