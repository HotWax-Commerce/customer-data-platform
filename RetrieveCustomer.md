FUNCTION retrieveCustomer(String partyId) RETURNS Map:

	// 1. Validate input
	IF empty(partyId):
		RETURN error("Party ID is required")

	// 2. Retrieve Party
	party = findOne("Party", {"partyId": partyId})
	IF party IS NULL:
		RETURN error("Customer not found")
		
	// 3. Retrieve Person record
	IF party.partyTypeId == 'PERSON' {
		person = findOne("Person", {"partyId": partyId})
	}

	// 4. Retrieve email Contact Mechanisms 
	emailContactMech = findFirst("PartyContactMech", {"partyId": partyId, "contactMechPurposeTypeId": "PRIMARY_EMAIL", "thruDate": null OR > presentTimeStamp})
	email = emailContactMech.infoString
    
    // 5. Retrieve Shipping Address
	contactMech = findFirst("PartyContactMech", {"partyId": partyId, "contactMechPurposeTypeId": "SHIPPING_LOCATION", "thruDate": null OR > presentTimeStamp})
	shippingAddress = findFirst("PostalAddress", {"contactMechId": contactMech.contactMechId})
	
	// 6. Retrieve Billing Address
	contactMech = findFirst("PartyContactMech", {"partyId": partyId, "contactMechPurposeTypeId": "BILLING_LOCATION", "thruDate": null OR > presentTimeStamp})
	billingAddress = findFirst("PostalAddress", {"contactMechId": contactMech.contactMechId})
	
	-> Address1, Address2, city, country will be obtained from the entity value of the postalAddress.
	
	// 7. Retrieve Telecom Number
	contactMech = findFirst("PartyContactMech", {"partyId": partyId, "contactMechPurposeTypeId": "PRIMARY_PHONE", "thruDate": null OR > presentTimeStamp})
	telecom = findFirst("TelecomNumber", {"contactMechId": contactMech.contactMechId})
	

	// 8. Additional Details like accepts marketing will be retrieved from PartyAttribute
	allow_marketing  = findOne("PartyAttribute", {"partyId": partyId, "attrName": "accepts_marketing"})
	
	// 9. Whether the customer is interested in Email or SMS marketing
	IF (allow_marketing) {
		partyClassificationAppl = findList("PartyClassificationAppl" , {"partyId": partyId})
		marketingConsentList = []
		FOR each pcal IN partyClassificationAppl:
			marketingConsentList.add(pcal.classificationTypeId)
	}
	IF (marketingConsentList contains "email_marketing_consent") 
		emailMarketingConsent = "subscribed"
		
	IF (marketingConsentList contains "sms_marketing_consent") 
		smsMarketingConsent = "subscribed"
		

	// 10. Assemble Result
	customer = makeValue("Customer", {
		"partyId": party.partyId,
		"externalId": party.externalId,
		"statusId": party.statusId,
		"firstName": person.firstName,
		"lastName": person.lastName,
		"birthDate": person.birthDate,
		"gender": person.gender,
		"email": email,
		"shippingAddress": shippingAddress,
		"billingAddress": billingAddress,
		"telecom": telecom,
		"emailMarketingConsent": emailMarketingConsent,
		"smsMarketingConsent": smsMarketingConsent
	})

	RETURN customer

