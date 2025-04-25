The "before closing" journal has a few foreign-currency transactions accumulating revenue that will be closed by `close --retain`. The total revenue is 500.00 CAD, 100.00 EUR, 1,000.00 SEK valued at 910.00 CAD as at the Day of Closing.

# `close --retain`

Now when closing the Income Statement accounts to Retained Earnings, we have two choices: CAD or native currency.

## Close in CAD

When we close to CAD, `close --retain --value="then,CAD"` generates a posting that fails the balance assertion, since although the total balance in the account is 0, the CAD balance is not 0.

```
2025-04-24 closing balances  ; start:
    Assets                          -910 CAD = 0 CAD
    Equity:Retained Earnings
```

Now I can't generate an income statement, because of the balance assertion error.

```
hledger: Error: /home/jbrains/Workspaces/hledger-foreign-balance-example/close_retained_earnings.CAD.journal:2:46:
  | 2025-04-24 retain earnings  ; retain:
2 |     Revenue                       910.00 CAD = 0.00 CAD
  |                                              ^^^^^^^^^^
  |     Equity:Retained Earnings     -910.00 CAD

Balance assertion failed in Revenue
In commodity CAD at this point, excluding subaccounts, ignoring costs,
the asserted balance is:          0.00 CAD
but the calculated balance is:  910.00 CAD
(difference: -910.00 CAD)
To troubleshoot, check this account's running balance with assertions disabled, eg:
hledger reg -I 'Revenue$' cur:CAD
```

but the balance of Revenue is -410.00 CAD, 100.00 EUR (@@ 110 CAD), 1,000.00 SEK (@@ 300 CAD) = 0.

In my opinion, this should pass _some kind of_ balance assertion. As far as the CAD equivalent is concerned, the balance of Revenue is 0.

## Close to native currencies

When we close to native currencies, `close --retain [no --value switch]` generates a posting that fails the balance assertion.

```
2025-04-24 retain earnings  ; retain:
    Revenue                       500.00 CAD = 0.00 CAD
    Revenue                          100 EUR = 0 EUR
    Revenue                       1,000. SEK = 0 SEK
    Equity:Retained Earnings
```

```
hledger: Error: /home/jbrains/Workspaces/hledger-foreign-balance-example/close_retained_earnings.FX.journal:2:46:
  | 2025-04-24 retain earnings  ; retain:
2 |     Revenue                       500.00 CAD = 0.00 CAD
  |                                              ^^^^^^^^^^
  |     Revenue                          100 EUR = 0 EUR
  |     Revenue                       1,000. SEK = 0 SEK
  |     Equity:Retained Earnings     -500.00 CAD
  |     Equity:Retained Earnings        -100 EUR
  |     Equity:Retained Earnings     -1,000. SEK

Balance assertion failed in Revenue
In commodity CAD at this point, excluding subaccounts, ignoring costs,
the asserted balance is:          0.00 CAD
but the calculated balance is:  500.00 CAD
(difference: -500.00 CAD)
To troubleshoot, check this account's running balance with assertions disabled, eg:
hledger reg -I 'Revenue$' cur:CAD
```

This, even though the income statement shows a balance of 0.00 CAD when reporting without converting currency.

```
$ hledger --file before_closing.journal --file close_retained_earnings.FX.journal incomestatement --ignore
Income Statement 2024-12-31..2025-04-24

          || 2024-12-31..2025-04-24 
==========++========================
 Revenues ||                        
----------++------------------------
----------++------------------------
          ||                      0 
==========++========================
 Expenses ||                        
----------++------------------------
----------++------------------------
          ||                      0 
==========++========================
 Net:     ||                      0 

```

And, of course, when we convert to CAD, Revenue now has a non-zero balance, due to the timing differences which normally would go to "Exchange Gain or Loss".

```
$ hledger --file before_closing.journal --file close_retained_earnings.FX.journal incomestatement --ignore --value="then,CAD"
Income Statement 2024-12-31..2025-04-24, valued at posting date

          || 2024-12-31..2025-04-24 
==========++========================
 Revenues ||                        
----------++------------------------
 Revenue  ||            -240.00 CAD 
----------++------------------------
          ||            -240.00 CAD 
==========++========================
 Expenses ||                        
----------++------------------------
----------++------------------------
          ||                      0 
==========++========================
 Net:     ||            -240.00 CAD 
```

Even worse, Retained Earnings is wrong, because of this timing difference in the currency value.

```
$ hledger --file before_closing.journal --file close_retained_earnings.FX.journal balancesheetequity --ignore --value="then,CAD"
Balance Sheet With Equity 2025-04-24, valued at posting date

                          ||   2025-04-24 
==========================++==============
 Assets                   ||              
--------------------------++--------------
 Assets                   ||   910.00 CAD 
--------------------------++--------------
                          ||   910.00 CAD 
==========================++==============
 Liabilities              ||              
--------------------------++--------------
--------------------------++--------------
                          ||            0 
==========================++==============
 Equity                   ||              
--------------------------++--------------
 Equity:Retained Earnings || 1,150.00 CAD 
--------------------------++--------------
                          || 1,150.00 CAD 
==========================++==============
 Net:                     ||  -240.00 CAD 
```

So... what's the right move here? Retained Earnings should be 910.00 CAD, but then I have to ignore the balance assertions, because I want to assert that _the converted balance_ of Revenue is 0 CAD and not that the CAD balance of Revenue is 0 CAD.

## Convert the Balance to 0 CAD!

How do I convert the balance of Revenue to 0 CAD without hardcoding currency exchange rates, which I expect would start a slow descent into madness?

