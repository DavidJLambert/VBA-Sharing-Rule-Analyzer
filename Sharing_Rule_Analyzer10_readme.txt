Sharing Rule Analyzer for Traveler 10.x and 11.x.

Created 07/08/99 by David Lambert.
Modified 09/10/99 to add check another check for malformed derived sharing rules.
Modified 11/01/02 to update for CeFO 10.x and 11.x.

Caveats:
This is not a QA'ed product.  The results are not guaranteed.  You are advised to 
double-check any errors this program identifies.

Instructions:
1)  Open up database in Access 2000.
2)  On the left hand side of the screen will be a column of icons.  Press the 
    "Forms" icon.
3)  Open up the "Sharing Rule Analyzer for CeFO 10" form.
4)  Follow the instructions on the form.

A note: this program depends on each sharing rule having a unique rule_id.
If the program shows missing or nonunique numbers, import de_share_rule if you 
linked it, fix the rule_id problem, and rerun the program.

If you do not know how to import or link tables, here's how.
(A link is basically a shortcut to a table in another database).

1)  Set up an ODBC data source pointing to the Traveler database.
2)  Go into the Tools menu and select Options.
3)  Select the View tab, and make sure the System Objects checkbox is selected.
4)  To export or link, go into the File menu and choose "Get External Data".
5)  Choose "Link Tables" or "Import".
6)  In the "Files of Type:" combo box, scroll to the bottom and select "ODBC Databases ()".
7)  Select the "Machine Data Source" tab, and highlight the Traveler database's data source.
8)  Click OK, and enter the login information at the prompt.
9)  A list of tables will pop up.  All of the tables will probably be prepended with "dbo.", 
    the owner of the tables.  Ignore whatever is prepended.
10) From the list, select de_share_rule, de_distrib_spec, de_rule_set, de_sr_alias,  
    adp_sch_rel_info, and adp_tbl_name_map.  Also select sysobjects (MSSQL) or user_tables 
    (Oracle).  Then press OK.
11) Skip any prompts to enter a unique key for a table.
12) Go into the Tools menu and select Options.
13) Select the View tab, and restore the System Objects checkbox to its original setting.
14) When linking to or importing from Oracle, sometimes some of the numerical fields will be 
    given a text datatype (text in Access means varchar elsewhere).  If this happens, the 
	sharing rule analyzer will not function properly.  In Access, you will have to open up 
	de_rule_set, de_share_rule, and de_distrib_spec in design view, and you will have to 
	change these fields to the "Number" datatype: rule_set, rule_id, distrib_order, and 
	share_type.

Icons for the tables will appear in the Tables tab, with names "<Owner Name>_<Table Name>".

You can leave the icon names as is, or change them to anything, since this 
utility selects the lists of tables based solely on whether or not they have 
fields that match the expected names of the fields in de_share_rule and 
de_table_bind.

This program does not make all possible checks on sharing rules.  Instead, it only checks 
for conditions that do not cause the sharing rule execution to generate errors.  
These checks are:
1)  Look for sharing rules with the same value of rule_id.
2)  Look for sharing rules with rule_id null.
3)  Look in clause_detail for relationships pointing to the wrong table, e.g. 
    "table_case.case_owner2user = table_employee.objid" 
    (the corrected version is "table_case.case_owner2user = table_user.objid").
4)  Look in clause_detail for bad references to de_sfa_sub or de_supp_sub, e.g.
    "table_case.case_owner2user = de_sfa_sub.row_id and 
    de_sfa_sub.table_name = 'table_employee'"
    (the corrected version is "table_case.case_owner2user = de_sfa_sub.row_id and 
    de_sfa_sub.table_name = 'table_user'").
5)  Look in clause_detail for node_id set to something that is not a node_id, e.g.
    "de_comm.node_id = table_user.objid"
    (the corrected version is "de_comm.node_id = table_user.objid").
6)  Find sharing rules for tables that are not in de_distrib_spec (not shared).  
    These sharing rules are never executed.
7)  Search for sharing rules with non-shared dependent tables, e.g. 
    "... t1.objid = s.row_id and s.table_name = 'table_subcase'", but table_subcase 
    is not a shared table.  Such rules never distribute any records.  The sample 
    from_clause makes this sharing depend on table_subcase.
8)  Look for sharing rules that depend on a table that do not exist, e.g.
    "... and s.table_name = 'table_subbbcase'".  Such rules never distribute records.
9)  Look for tables with a sharing rule that depends on another table that 
    does not have a smaller distrib_order.  This causes sharing rules to execute 
    accross two runs of the Distributor.
10) Look for circularities in sharing rules, e.g. two or more sharing rules that 
    cause at least two tables to depend on each other.  All tables in such a 
    circularity will never be able to generate losses for some records.  This 
    program is only sophisticated enough to look for circularities involving 
    one "backwards" sharing rule found in item 9 above.  This program DOES find 
    circularities involving any number of daisy-chained "forwards" sharing rules, 
    and all possible daisy-chains are found and listed.
11) Look for derived sharing rules where S.ROW_ID is present but S.TABLE_NAME is absent, 
    or S.ROW_ID is absent but S.TABLE_NAME is present.
12) Look for sharing rules containing x.focus_type = N or x.focus_lowid = y.field, and checks 
    to see that if x.focus_type is present in the sharing rule only when x.focus_lowid is also 
    present and vice versa, checks that N is a valid type_id, and checks that N is the type_id
    of whatever y.field points to.

Checks 1 and 2 are only done because this program identifies sharing rules assuming 
that each sharing rule has a unique number or is the only sharing rule for that table 
without a number.

Warning: the mdb file containing this utility will gradually grow in size with repeated 
use, since it repeatedly adds new tables and adds records to these tables.  It is 
recommended that you periodically compact the database.  To compact the database, go 
into the Tools menu, select "Database Utilities", then select "Compact and Repair Database".

=============================================

Use Table Aliases to Eliminate Circular Rules

Another way to avoid the Circular Rule Trap is to create an alias for the table
with the higher distribution order, so that the table with the lower distribution 
order can use information about the table with the higher distribution order
without creating a circularity.

Alias tables have sharing rules, a rule set, a distribution order, subscriptions, 
a record in de_distrib_spec, and records in de_share_rule, 
just like regular tables.  However, there are four important ways
that alias tables differ from regular tables:
1)  The record in de_distrib_spec has share_type=3.
2)  The alias table does not actually exist in the Clarify database.
3)  Gains and losses are not distributed for changes in subscriptions to alias tables.
4)  Subscriptions to alias tables are not merged.  Thus, for an alias table in 
a non-universal rule set (for example, the SFA rule set), subscriptions to the alias 
table are written into de_sfa_sub, but are not copied to de_subscribe for merging.

Therefore, an alias functions as a temporary storage depot for subscriptions, solely 
for the use of sharing rules for other tables.  These alias table subscriptions stand 
in for subscriptions of the table being aliased.

Let's use the previous example to illustrate aliases and how they are used.  Create an 
alias for subcase, called subcase_alias, with distribution order 10.  
Thus, we have three tables, two of them real, and one an alias table:
	Subcase_alias, with distribution order 10.
	Case, with distribution order 20.
	Subcase, with distribution order 25.

Here is one way to create sharing rules for the alias table to avoid a circularity.
	Sharing rules for Subcase_alias, distribution order 10:
		Rule A1: If you subscribe to a subcase, you subscribe to a subcase_alias with the same name.
	Sharing rules for Case, distribution order 20:
		Rule C1: Distribute cases to case owners.
		Rule C2: Distribute cases to owners of related subcases in subcase_alias.
	Sharing rules for Subcase, distribution order 25:
		Rule S1: Distribute subcases to subcase owners.
In this method, there is one derived sharing rule, A1, for subcase_alias that looks to subcase, 
which has a higher distribution order.  This means it takes two executions of the distributor to 
finish executing all these sharing rules.  Thus, this method is not recommended.

Here is the other way to create sharing rules for the alias table to avoid a circularity.
	Sharing rules for Subcase_alias, distribution order 10:
		Copies of the sharing rules for subcase, with subcase_alias substituted in place of subcase.
	Sharing rules for Case, distribution order 20:
		Rule C1: Distribute cases to case owners.
		Rule C2: Distribute cases to owners of related subcases in subcase_alias.
	Sharing rules for Subcase, distribution order 25:
		Rule S1: Distribute subcases to subcase owners.
This method does not have derived sharing rules looking to tables with a higher distribution order, 
so only one execution of the distributor is required.  Thus, this method is the recommended method.

Here is what the SQL for a rule that distributes open cases to nodes that subscribe to
associated sites looks like:
SELECT DISTINCT T0.OBJID, S.MAP1, S.MAP2, S.MAP3,
S.MAP4, S.MAP5, S.MAP6, S.MAP7
FROM TABLE_CASE T0, TABLE_CONDITION T1, DE_SFA_SUB S
WHERE T0.CASE_STATE2CONDITION = T1.OBJID AND T1.TITLE =
''OPEN'' AND T0.CASE_REPORTER2SITE = S.ROW_ID AND
S.TABLE_NAME = ''TABLE_SITE''

Suppose we have an alias table for table_site, named table_site_alias.  The same rule that uses this
alias for table_site would use the following SQL code:
SELECT DISTINCT T0.OBJID, S.MAP1, S.MAP2, S.MAP3,
S.MAP4, S.MAP5, S.MAP6, S.MAP7
FROM TABLE_CASE T0, TABLE_CONDITION T1, DE_SFA_SUB S
WHERE T0.CASE_STATE2CONDITION = T1.OBJID AND T1.TITLE =
''OPEN'' AND T0.CASE_REPORTER2SITE = S.ROW_ID AND
S.TABLE_NAME = ''TABLE_SITE_ALIAS''
Note that even though we are looking to table_site_alias for subscriptions, we are using 
case_reporter2site, a relation that connects table_case to table_site, to "walk" from 
table_case to table_site_alias.


To create a table alias:
1. Log in to the Traveler Administration Tools.
2. Choose "Tools" -> "Table Alias".
3. Press the "Add Alias" button.
4. Enter the actual database table name in the "Table Name" field, omitting "Table_".
5. Enter a substitute name for the table in the "Alias Name" field.
6. Add a description that describes the reason for the alias.
7. Click Apply.
8. Click Close.

This only creates the alias.  You still have to enter the alias table in de_distrib_spec.  
To do this, follow the usual instructions for placing a new table into a rule set, using the 
alias table name.

Also, you have to create sharing rules for the alias table.  Follow the usual instructions 
for creating sharing rules, using the alias table name.  Be sure to set "Share Type" to "Alias".  

You will be creating at least one new derived sharing rule for some table that points 
to the alias table, rather than to the table being aliased.  Start by entering the sharing rule 
the usual way.  If this is a standard rule, when you press Add or Edit for the part of the new 
rule that points to the alias table, enter the name of the relation to the table being aliased.
Then select the alias table, rather than the table being aliased.  The table dropdown list box will 
list both the alias table and the table being aliased as options.

Finally, compile the rules and export them the usual way.