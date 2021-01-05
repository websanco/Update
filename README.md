This is never really a good position to be in, but there are ways of rebuilding these files into a clean slate.
Note, that we recommend using:
innodb_file_per_table

in the [mysqld] section of your /etc/my.cnf from install time.  Changing it after databases already exist could break things and force you to use this guide.
If you want to change to use innodb_file_per_table, then you'd need to use this guide.
It will essentially dump all databases to sql files, drop all databases, stop mysqld, delete the ib* data files, and restart mysqld and restore the sql files.

1) Before anything else, ensure you have full .sql backups of all of your data.  It's also recommended to have a copy of the mysql data directory, so you can copy it back if things go south.

CustomBuild 1.2 and 2.0 can make the plaintext sql files for you.  Run the following, but only if your data itself is still valid. Do not run this if you need to restore your backups to get non-corrupted data:
cd /usr/local/directadmin/custombuild
./build update
./build set mysql_backup yes
./build mysql_backup

Then confirm that the sql files have been created here:
ls -la /usr/local/directadmin/custombuild/mysql_backups

If you do not have any valid plain .sql file backups, do not proceed.  This guide only works if if you have something to restore from.

Note: The innodb_force_recovery option may let you get mysqld running, and possibly even repair it for you in a quick/easy manner.


2) Next, we'll shut off mysqld to make a data directory backup, which you'll want to do either way (corrupted or not).
perl -pi -e 's/mysqld=ON/mysqld=OFF/' /usr/local/directadmin/data/admin/services.status
/etc/init.d/mysqld stop

then copy the directory.  Use /home/mysql for Debian or FreeBSD.
cd /var/lib
cp -Rp mysql mysql.innodb.backup



3a) Try this guide to attempt to surgically fix or remove the offending database:
http://blackbird.si/mysql-corrupted-innodb-tables-recovery-step-by-step-guide/
Hopefully this will work for you, and you can stop without a full rebuilt.
If this fixes the problem, stop here, do not continue.

3b) At this point, assuming you have safe backups of all of your data, you can move to the point of no return.
Start mysqld again, and drop all databases, except these three: mysql, performance_schema, information_schema.
To do this, you can use /phpMyAdmin via Apache to make things easier and quicker.
Use the da_admin user/pass from:
cat /usr/local/directadmin/conf/mysql.conf

Drop all, but those 3 databases.

4) Now that they're gone, we can clean up the ib* data.   Stop mysqld again, delete the ib* files, and start mysqld. If you are changing to use the innodb_file_per_table option, make sure it's in your /etc/my.cnf at this point.
cd /var/lib/mysql
/etc/init.d/mysqld stop
rm -f ib*
/etc/init.d/mysqld start
perl -pi -e 's/mysqld=OFF/mysqld=ON/' /usr/local/directadmin/data/admin/services.status

this will rebuild the ibdata1 and innodb log files, goodbye corruption.

5) Last step is to restore your sql data files.  Because the mysql database was left, the users should all still be there (it's tables all use MyISAM, thus not affected by innodb corruption). Assumign your sql files are in the path above, then you should be able to run the following script:
cd /usr/local/directadmin/custombuild/mysql_backups
wget http://files1.directadmin.com/services/all/mysql/restore_sql_files.sh
chmod 755 restore_sql_files.sh
./restore_sql_files.sh

This will automatically skip the mysql and schema databases, and restore everything else.
