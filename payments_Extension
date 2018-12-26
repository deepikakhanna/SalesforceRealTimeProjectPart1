public with sharing class payments_Extension {
    //Payment__c record variable to hold payment information
    public Payment__c thisPayment {get;set;}
    public Statement__c thisStatement {get;set;}
    public map<id, string> recordTypeMap {get;set;}

    //boolean to determine if payment was successful
    public boolean success {get;set;}

    public payments_Extension(ApexPages.StandardController scon) {
        if (scon.getRecord() instanceof Statement__c) thisStatement = [select id, name, Rental_Agreement__r.renter__c, balance__c from Statement__c where id = :scon.getid()];
        success = false;
        recordTypeMap = new map<id,string>();
        //query for all payment record types
        //populate map
        for (RecordType r : [select id, name from RecordType where sobjecttype='Payment__c']){
            recordTypeMap.put(r.id,r.name);
        }

        //instantiate payment
        thisPayment = new Payment__c();
        if (scon.getRecord() instanceof Statement__c){
            thisPayment.Statement__c = scon.getid();
            thisPayment.Amount__c = thisStatement.balance__c;
        }

        //if guest user, preset type to credit card
        if (getIsGuest()){
            for (id i : recordTypeMap.keyset()){
                if (recordTypeMap.get(i) == 'Credit Card'){
                    thisPayment.recordtypeid = i;
                    break;
                }
            }
        }
        //if not guest, populate billing details from renter
        else if (scon.getRecord() instanceof Statement__c){
            Contact renter = [select id, firstname, lastname, mailingstreet, mailingcity,mailingstate,mailingpostalcode from Contact where id =:thisStatement.Rental_Agreement__r.renter__c];
            thisPayment.Billing_Name__c = renter.firstname+' '+renter.lastname;
            thisPayment.Billing_Street__c = renter.mailingstreet;
            thisPayment.Billing_City__c = renter.mailingcity;
            thisPayment.Billing_State__c = renter.mailingstate;
            thisPayment.Billing_Postal_Code__c = renter.mailingpostalcode;
        }
    }

    //this method will process and save our payment
    //or report any errors in the attempt
    public pagereference savePayment(){
        success = false;
        string paymentType = recordTypeMap.get(thisPayment.recordtypeid);
        //check if all fields filled out
        if (validateFields(paymentType)){
            //process credit card payments
            if (paymentType == 'Credit Card'){
                //create a request wrapper for authorize.net
                api_AuthorizeDotNet.authNetReq_Wrapper req = new api_AuthorizeDotNet.authNetReq_Wrapper();

                //set wrapper values
                req.amt = string.valueof(thisPayment.Amount__c);
                req.firstname = (thisPayment.Billing_Name__c.contains(' '))?thisPayment.Billing_Name__c.substringbefore(' '):thisPayment.Billing_Name__c;
                req.lastname = (thisPayment.Billing_Name__c.contains(' '))?thisPayment.Billing_Name__c.substringafter(' '):thisPayment.Billing_Name__c;
                req.billstreet = thisPayment.Billing_Street__c;
                req.billcity = thisPayment.Billing_City__c;
                req.billstate = thisPayment.Billing_State__c;
                req.billzip = thisPayment.Billing_Postal_Code__c;

                //set wrapper credit card fields
                req.ccnum = thisPayment.Credit_Card_Number__c;
                req.ccexp = monthmap.get(thisPayment.Credit_Card_Expiration_Month__c)+thisPayment.Expiration_Year__c;
                req.ccsec = thisPayment.Credit_Card_Security_Card__c;

                //give this request a name
                req.ordername = 'Payment of '+[select id,name from Statement__c where id = :thisPayment.Statement__c].name;
                //process authorize.net request
                api_AuthorizeDotNet.authNetResp_Wrapper res = api_AuthorizeDotNet.authdotnetCharge(req);

                //authorize.net data regarding the transaction
                thisPayment.Authorize_net_Transaction_ID__c = res.TransactionID;
                thisPayment.Authorize_net_Authorization_Code__c = res.AuthorizationCode;
                thisPayment.Authorize_net_Response__c = res.ResponseCode+'| '+res.ResponseReasonText; 

                //if the transaction failed
                if (res.responseCode != '1' || res.ResponseReasonText != 'This transaction has been approved.'){
                    thisPayment.Status__c = 'Failed';
                    ApexPages.addMessage(new ApexPages.message(ApexPages.severity.error,'Payment Failed'));
                    ApexPages.addMessage(new ApexPages.message(ApexPages.severity.error,'res.ResponseReasonText'));
                    return null;
                }
            }
            
            //successful transactions
            thisPayment.Status__c = 'Paid';
            thisPayment.Payment_Date__c = system.now();
            upsert thisPayment;
            success = true;
            ApexPages.addMessage(new ApexPages.message(ApexPages.severity.confirm,'Payment Successful'));

            try{
                //if there is an email on the contact, send them a confirmation email
                if (thisstatement !=null && thisstatement.Rental_Agreement__r.renter__r.email != null){
                    //construct message
                    Messaging.SingleEmailMessage msg = new Messaging.SingleEmailMessage();
                    //to addresses is a list
                    msg.setToAddresses(new list<string>{thisstatement.Rental_Agreement__r.renter__r.email});
                    msg.setsubject('Payment Confirmation');
                    //you can set both html and plaintext bodies in case the 
                    //recipient does not receive html
                    msg.setHTMLbody('Your payment of '+thisPayment.Amount__c+' has been successfully processed.<br/><br/>Thank you.');
                    msg.setplaintextbody('Your payment of '+thisPayment.Amount__c+' has been successfully processed.\n\nThank you.');
                    //send the email
                    Messaging.sendEmail(new list<Messaging.SingleEmailMessage>{msg});
                }
            }
            catch(exception e){}
        }
        else {
            ApexPages.addMessage(new ApexPages.message(ApexPages.severity.error,'Please fill out all the details'));
        }
        return null;
    }

    //verify required fields have been filled out
    public boolean validateFields(string paymentType){
        boolean valid = true;

        //check common fields
        if (thisPayment.statement__c==null) valid = false;
        if (thisPayment.Amount__c==null) valid = false;
        if (string.isBlank(thisPayment.Billing_Name__c)) valid = false;
        if (string.isBlank(thisPayment.Billing_Street__c)) valid = false;
        if (string.isBlank(thisPayment.Billing_City__c)) valid = false;
        if (string.isBlank(thisPayment.Billing_State__c)) valid = false;
        if (string.isBlank(thisPayment.Billing_Postal_Code__c)) valid = false;

        //check specific fields
        if (paymentType  == 'Check'){
            if (string.isBlank(thisPayment.Check_Account_Number__c)) valid = false;         
            if (string.isBlank(thisPayment.Check_Routing_Number__c)) valid = false;         
        }
        //credit card specific fields
        else if (paymentType == 'Credit Card'){
            if (string.isBlank(thisPayment.Credit_Card_Number__c)) valid = false;           
            if (string.isBlank(thisPayment.Credit_Card_Expiration_Month__c)) valid = false;
            if (string.isBlank(thisPayment.Expiration_Year__c)) valid = false;
            if (string.isBlank(thisPayment.Credit_Card_Security_Card__c)) valid = false;
        }

        return valid;
    }

    //getter method for guest user determination
    public boolean getIsGuest(){
        //match profile based on current user
        return [
                select id, userType 
                from Profile 
                where id = :userInfo.getProfileId()
            ].userType == 'Guest';
    }

    //return select options based on class map
    public list<selectoption> getPaymentRecordTypes(){
        list<selectoption> temp = new list<selectoption>();
        //select option structure is value, label, disabled (optional)
        temp.add(new selectoption('','Select Payment Method'));
        for (id i : recordTypeMap.keyset()){
            temp.add(new selectOption(i,recordTypeMap.get(i)));
        }
        return temp;
    }

    //construct select options for this year
    //and the next 4 years after that
    public list<selectOption> getExpirationYears(){
        list<selectoption> temp = new list<selectoption>();
        for (integer i=0; i<5; i++){
            string y = ''+system.today().addyears(i).year();
            temp.add(new selectoption(y,y));
        }
        return temp;
    }

    //map to convert months to 2 digits for authorize.net
    public static map<string,string> monthMap = new map<string,string>{
        'January'=>'01',
        'February'=>'02',
        'March'=>'03',
        'April'=>'04',
        'May'=>'05',
        'June'=>'06',
        'July'=>'07',
        'August'=>'08',
        'September'=>'09',
        'October'=>'10',
        'November'=>'11',
        'December'=>'12'
    };
}
