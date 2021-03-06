# System description

William Sexton

7/19/2017

## Conceptual outline:

There are two main phases:

1. Bayesian bootstrapping on housing files

	a) Housing units

	b) Group Quarters

2. Populating housing units with person records

### Phase 1 description:

**Goal:** Draw N samples from a dirichlet-multinomial distribution over all the records in the US housing file.
      
**Defining the distribution:** We need to compute wgt(i)*n_yr(i)/sum_i wgt(i) for each record i where wgt(i) is the housing weight of i and n_yr(i) is the number of records in the US housing file from the same year as record i.
      
**Challenge:** Group quarters records in the housing file do not have housing weights.

**Current Solution:** Bootstrap the housing units and group quarters units separately.
 
a) Housing Units:
 
 &nbsp;&nbsp;&nbsp;&nbsp;Let N_h be the target number of housing units for the synthesize data. N_h is hardcoded now--should be changed to config file.

&nbsp;&nbsp;&nbsp;&nbsp;Let wgt(i) be the housing weight of housing unit i.

&nbsp;&nbsp;&nbsp;&nbsp;Let n_yr(i)^h be the total number of housing units in the US from the same year as housing unit i.

&nbsp;&nbsp;&nbsp;&nbsp;Note that sum_i wgt(i) is summed over all housing units in the US.

&nbsp;&nbsp;&nbsp;&nbsp;Let n_h be the total number of housing unit records in the housing files.

&nbsp;&nbsp;&nbsp;&nbsp;Let alpha_h be a vector of length n_h defined by (wgt(i)*n_yr(i)^h/sum_i wgt(i)). You need the whole US here.

&nbsp;&nbsp;&nbsp;&nbsp;Let theta_h=Dirichlet(alpha_h). You need the whole US here.

&nbsp;&nbsp;&nbsp;&nbsp;Let counts_h=Multinomial(N_h,theta_h). counts_h is essentially a histogram over housing units.  

&nbsp;&nbsp;&nbsp;&nbsp;In the synthetic data record i must be duplicated couns_h[i] times. You need the whole US for  
doing the Multinomial but after counts_h is computed it could be split up by state.

b) Group Quarters:

&nbsp;&nbsp;&nbsp;&nbsp;There are no weights on group quarters units in the housing files but each group quarters unit in the ACS is linked to exactly one person record so the idea is to indirectly bootstrap the group quarters units from the group quarters population.

&nbsp;&nbsp;&nbsp;&nbsp;Let N_g be the target group quarters population. N_g is hardcoded now--should be changed to config file. 

&nbsp;&nbsp;&nbsp;&nbsp;Let wgt(i) be the person weight of the person linked to the group quarters unit i.

&nbsp;&nbsp;&nbsp;&nbsp;Hence, sum_i wgt(i) is summing up all the person weights of person records linked to group quarters units.

&nbsp;&nbsp;&nbsp;&nbsp;Everything else is similar to above. counts_g gives a histogram over group quarters units that tells you how many times each group quarters unit must be duplicated.

&nbsp;&nbsp;&nbsp;&nbsp;The final part of this phase is to actually duplicate each record to create the synthetic housing data files.

### Phase 2 description:
      Goal: Populate the households/group quarters.
      Challenge illustrate by example: Suppose a housing unit with 3 occupants (mom,dad, 1 kid) got duplicated 10 times. The mom,dad, and kid each of the same serialno and it links them to all 10 housing units.
      Solution: Define new unique identifier for each copy of the housing unit. Create 10 copies of the family and link each to a unique housing unit using the new identifiers. 

Description of programs:
1. gen_counts.py
      Goal: Make the counts_h and counts_g histograms.
      Challenge: Defining alpha_h and alpha_g require aggragating information from all the US(both from housing files and from person files). Loading all the US housing files/person files into a single dataframe is a very bad idea. I tried it.
      Solution: Process data in chunks. Aggragate required information a chunk at a time. firstpass.py collects this information.
      Challenge: ACS doesn't include a year variable.
      Solution: Adjustment factors are specific to a year so I used ADJINC as a proxy for year but this required hardcoding the mapping between the adjustment factor and the year. This is one reason why the program won't handle the 2011-2015 data.
      Alternative Solution: I only recently realized that the first 4 digits of the serialno are always the year. This isn't explicitly stated in the documenation but seems pretty clear from looking at serialno range. I'm constantly learning new things about the input data. This seems like a more nature way to determine year, will require less hardcoding, and make the program more robust to input ranges. I plan on changing the code to this method.
      Challenge: Associating a count in counts_h or counts_g with a specific record.
      Solution: This is fairly straight forward. Index by serialno. Technically counts_g/h are pandas dataframes indexed by serialno with a single Count column.

Given the indexing choice, counts_g and counts_h can be safely concatenated into a single dataframe 'counts'.  This is the output from gen_counts.py. Alternatively, we could split 'counts' up by state.

2. rep_syn_housing.py
      Goal: Duplicate each housing/group quarters units according to the counts from gen_counts.py.
      Current Process:
         Read a chunk of housing records into a dataframe, join the counts dataframe onto the end, duplicate each row by the given count, add unique id, drop the counts column, write to an synthetic housing output file. Repeat.
      Alternative 1: Read in a single row from housing files, lookup count, duplicate, add unique identifier, write to output, repeat. I think the join process is more efficient but haven't run tests to validate.
      Alternative 2: Assuming counts is separated by state. Parallelize the current process by state. This should make the join operation more efficient but adding the identifier and writing the output will be trickier.
      Purpose of the dictionary: Explained in next section. 

3. rep_syn_person.py
      Goal: Duplicate person records and link each duplicated person to a unique household/group quarter.
      Challenge: Back to our 3 person household example-counts tells me that 10 copies of household with serialno (eg 20120000337) were made during rep_syn_housing.py but doesn't tell me what unique id was assigned to each copy. Hence, I know to make 10 copies of each person with serialno 20120000337 but I don't know how to link them to households.
      Current Solution: During rep_syn_housing.py, I create a dictionary where a key is a serialno and the corresponding value is a list of the unique ids associated with that serialno. Now when I create 10 copies of a person with serialno 20120000337 I can lookup the list of unique ids and give one id to each copy.
      Alternative Solution 1: Iterate through all the newly created synthetic housing csv files looking for unique ids associated with a serialno. Saves on memory but seems very inefficient.
      Alternative Sloution 3: Something without the dictionary. Is there another way to accomplish the same thing? 

      Current Process:
         Read a chunk of person records into a dataframe, join the counts column, duplicate persons, use dictionary to assign new ids, drop counts. Repeat. Chunks can and are currently parallel processed but the size of the dictionary makes things very slow.
      Alternative Process: Define separate dictionaries for each state in rep_syn_housing.py. Parallelize the current process across states. I'm pretty sure this will significantly boost runtime. Not sure it will change the memory requirements much.

Notes for e2e with RI,WV, WA:
      We need to change N_g and N_h otherwise we are trying to synthesize the entire US population from 3 states which doesn't make sense and also causes memory problems.
 
