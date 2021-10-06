

# Dashboard

![BaselineVsOptimized](https://user-images.githubusercontent.com/79966817/136270980-831deb75-1cd1-44b6-ad99-33292e40fb48.png)

![SignalsMain](https://user-images.githubusercontent.com/79966817/136271592-029f6d7c-c397-4772-b0fa-648db43f16ff.png)

![image](https://user-images.githubusercontent.com/79966817/136274124-d1a01862-76a0-43dd-bb89-811f765d345c.png)

![baseline v opt](https://user-images.githubusercontent.com/79966817/136274170-21f991fc-6920-48ca-8ae8-9f95063415f7.png)


# Sample SQL Code
### Reverse engineering default building settings without optimization for comparison 
``` 
SELECT * 
FROM (
  SELECT  d.DateCreatedUTC , 
          d.value as TempOutside 
  FROM Buildings b 
  INNER JOIN Signals s ON b.id=s.BuildingID 
  INNER JOIN Data d ON d.SignalID =s.ID 
  WHERE   b.name = 'Novi Building' AND 
          s.name = 't' AND 
          s.Enabled =1 AND 
          d.DateCreatedUTC > DATEADD(dd, -7, GETUTCDATE())
      ) as t
 OUTER APPLY (
   SELECT  t.id, 
   	   t.readName, 
           MAX(CalcSetPoint) as CalculatedSetPoint
    FROM (
    	SELECT *,
	       CASE WHEN TempOutside >= OutsideTemp AND TempOutside <= LeadOutsideTemp THEN FallbackValue + (TempOutside - OutsideTemp) / (LeadOutsideTemp - OutsideTemp) * (LeadFallback - FallbackValue) ELSE NULL	END as CalcSetPoint
    	FROM  (
	      	SELECT *,
		       LEAD(FallbackValue) OVER (PARTITION BY t.id ORDER BY idx ASC) as LeadFallback,
		       LEAD(OutsideTemp) OVER (PARTITION BY t.id ORDER BY idx ASC) as LeadOutsideTemp
   	      	FROM (
             	       SELECT  *,
		               (idx-1) / (MAX(idx) OVER (partition BY t.id)-1) Proportion,
		     		-20 + (40*(idx-1) / (MAX(idx) OVER (partition BY t.id)-1)) OutsideTemp
	      	       FROM (
                  		SELECT  sr.id, 
                          		sr.Name as readName,
				        sw.Name as writeName,
			                sw.FallbackValue,
				        CAST(RIGHT(sw.Name, 1) as float) as idx
				        FROM Signals sr
				        INNER JOIN Transformation t  ON t.SignalParentID =sr.ID 
				        INNER JOIN Signals sw ON sw.id = t.SignalID
				        INNER JOIN Buildings b ON b.id=sr.BuildingID 
			          	WHERE sr.rw='r' AND 
                       			      sr.Name LIKE '%vs%'AND 
                        		      sr.Enabled =1 AND
                        		      sw.Name LIKE '%[_]Y_' AND
                       			      b.name = '******'
			     ) as t
		    ) as t
	       ) as t
       ) as t
   	GROUP BY t.id, t.readName
 ) as t2̈́
ORDER BY DateCreatedUTC DESC
```
