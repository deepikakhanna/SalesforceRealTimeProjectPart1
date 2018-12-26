public class Rollup_Methods {
    public static void rollupStatementsToRentalAgreement(List<Statement__c> newList)
    {
        Set<id> rentalAgreementSet=new Set<id>();
        for(Statement__c s:newList)
        {
            rentalAgreementSet.add(s.Rental_agreement__c);
            
        }
        //this list will store the rental Agreement records which needs to be updates
        List<Rental_Agreement__c> rentalUpdates=new List<Rental_Agreement__c>();
        //aggregate query to sum total amount from statement
        
        for(AggregateResult ar:[select sum(Total_Amount__c)totalInvoice,rental_agreement__c from Statement__c where Rental_Agreement__c IN :rentalAgreementSet group by Rental_Agreement__c])
        {
            Rental_Agreement__c r=new Rental_Agreement__c(id=String.valueOf(ar.get('rental_agreement__c')));
            r.Total_Invoiced__c=double.valueOf(ar.get('totalInvoice'));
            rentalUpdates.add(r);
        }
        if(!rentalUpdates.isEmpty())
        {
            update rentalUpdates;
        }
    }

     public static void rollupPaymentsToRentalAgreement(Map<id,Payment__c> newMap)
     {
         Set<Id> rentalAgreementSet =new Set<id>();
         for(Payment__c p : [select id,statement__r.Rental_Agreement__c from Payment__c where Id IN :newMap.keySet()])
         {
             rentalAgreementSet.add(p.statement__r.Rental_Agreement__c);
         }
         List<Rental_Agreement__c> rentalUpdates=new List<Rental_Agreement__c>();
         for(AggregateResult ar:[select sum(Amount__c)totalPaid,Statement__r.Rental_Agreement__c ra from Payment__c where Statement__r.Rental_Agreement__c IN :rentalAgreementSet group by Statement__r.Rental_Agreement__c  ])
         {
             Rental_Agreement__c r=new Rental_Agreement__c(id=string.valueOf(ar.get('ra')));
             r.Total_Payments__c=double.valueOf(ar.get('totalPaid'));
             rentalUpdates.add(r);
         }
         if(!rentalUpdates.isEmpty())
         {
             update rentalUpdates;
         }
    }
}
