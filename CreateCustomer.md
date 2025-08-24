FUNCTION createCustomer(Delegator delegator, Map context) RETURNS Map:

    // 1. Validate input
    IF empty(context.firstName) OR empty(context.email):
        RETURN error("First Name and Email are required")

    IF not validEmail(context.email):
        RETURN error("Invalid email format")

    partyId = delegator.getNextSeqId("Party")

    // 2. Create PARTY + PERSON
    IF (context.partyTypeId == "PERSON") {
        party = delegator.makeValue("Party", {
            "partyId": partyId,
            "partyTypeId": "PERSON",
            "externalId": context.externalId,
            "statusId": "PARTY_ENABLED",
            "createdDate": now(),
            "lastUpdatedDate": now()
        })
        party.store()

        person = delegator.makeValue("Person", {
            "partyId": partyId,
            "firstName": context.firstName,
            "lastName": context.lastName,
            "birthDate": context.birthDate,
            "gender": context.gender
        })
        person.store()
    }

    // 3. Create PARTY_ROLE (Customer)
    partyRole = delegator.makeValue("PartyRole", {
        "partyId": partyId,
        "roleTypeId": "CUSTOMER"
    })
    partyRole.store()

    // 4. Create EMAIL Contact Mechanism
    IF not empty(context.email):
        emailContactMechId = delegator.getNextSeqId("ContactMech")
        contactMech = delegator.makeValue("ContactMech", {
            "contactMechId": emailContactMechId,
            "contactMechTypeId": "EMAIL_ADDRESS",
            "infoString": context.email
        })
        contactMech.store()

        partyContactMech = delegator.makeValue("PartyContactMech", {
            "partyId": partyId,
            "contactMechId": emailContactMechId,
            "contactMechPurposeTypeId": "PRIMARY_EMAIL",
            "fromDate": now()
        })
        partyContactMech.store()

    // 5. Create POSTAL ADDRESS
    IF not empty(context.address1):
        addressContactMechId = delegator.getNextSeqId("ContactMech")
        contactMechAddr = delegator.makeValue("ContactMech", {
            "contactMechId": addressContactMechId,
            "contactMechTypeId": "POSTAL_ADDRESS"
        })
        contactMechAddr.store()

        postalAddress = delegator.makeValue("PostalAddress", {
            "contactMechId": addressContactMechId,
            "toName": context.firstName + " " + context.lastName,
            "address1": context.address1,
            "address2": context.address2,
            "city": context.city,
            "state": context.state,
            "country": context.country,
            "postalCode": context.postalCode
        })
        postalAddress.store()

        partyContactAddr = delegator.makeValue("PartyContactMech", {
            "partyId": partyId,
            "contactMechId": addressContactMechId,
            "contactMechPurposeTypeId": "PRIMARY_ADDRESS",
            "fromDate": now()
        })
        partyContactAddr.store()

    // 6. Create TELECOM (Phone)
    IF not empty(context.phoneNumber):
        phoneContactMechId = delegator.getNextSeqId("ContactMech")
        contactMechPhone = delegator.makeValue("ContactMech", {
            "contactMechId": phoneContactMechId,
            "contactMechTypeId": "TELECOM_NUMBER"
        })
        contactMechPhone.store()

        telecom = delegator.makeValue("Telecom", {
            "contactMechId": phoneContactMechId,
            "countryCode": context.countryCode,
            "areaCode": context.areaCode,
            "number": context.phoneNumber
        })
        telecom.store()

        partyContactPhone = delegator.makeValue("PartyContactMech", {
            "partyId": partyId,
            "contactMechId": phoneContactMechId,
            "contactMechPurposeTypeId": "PRIMARY_PHONE",
            "fromDate": now()
        })
        partyContactPhone.store()

    // 7. Create PARTY ATTRIBUTES (accepts marketing, opt-in level)
    partyAttr1 = delegator.makeValue("PartyAttribute", {
        "partyId": partyId,
        "attrName": "accepts_marketing",
        "attrValue": context.accepts_marketing
    })
    partyAttr1.store()

    partyAttr2 = delegator.makeValue("PartyAttribute", {
        "partyId": partyId,
        "attrName": "marketing_opt_in_level",
        "attrValue": context.marketing_opt_in_level
    })
    partyAttr2.store()

    // 8. Create PARTY CLASSIFICATION APPL (Marketing Consent)
    IF context.accepts_marketing == "Y":
        IF context.email_marketing_consent.state == "subscribed":
            partyClassificationAppl = delegator.makeValue("PartyClassificationAppl", {
                "partyId": partyId,
                "partyClassificationId": "EMAIL_MARKETING_CONSENT",
                "fromDate": now()
            })
            partyClassificationAppl.store()

        IF context.sms_marketing_consent.state == "subscribed":
            partyClassificationAppl = delegator.makeValue("PartyClassificationAppl", {
                "partyId": partyId,
                "partyClassificationId": "SMS_MARKETING_CONSENT",
                "fromDate": now()
            })
            partyClassificationAppl.store()

    RETURN success("Customer created successfully", {"partyId": partyId})

