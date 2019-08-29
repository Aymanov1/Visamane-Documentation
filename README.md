# otome-backend


<h1 align="center">
  <br>
  <img src="batch.png" height="200" width="200"/>
  <br>
  Otome(Batch server) Backend
  <br>


<hr>
<p>Otome(Batch server) is a  SpringBoot app to manage the autocrawlring and the auto injection processes of jobs from differents websites <strong>(Baitoru, Lacotto, JSEN)</strong>.</p>

<br><br>
## Tools

<ul>
<li> SpringBoot </li>
<li> Crawler4j </li>
<li> Jsh </li>
<li> Spring_Security </li>
<li> Reactor </li>
<li> Swagger2 </li>
<li> OpenCSV </li>
<li> SuperCSV </li>
</ul>

<br><br>
## New Features

<ul>
<li>  <a href="https://spring.io/guides/gs/accessing-data-rest/">HATEOAS API</a></li>
<li>  <a href="https://github.com/Aymanov1/telegram-community-backend/tree/master/src/main/resources">Spring profiles</a> </li>
<li>  <a href="https://spring.io/projects/spring-security">Spring Security</a></li>
<li>  <a href="https://cloud.google.com/natural-language/?hl=en">Cloud Language API</a></li>
<li>  <a href="https://www.baeldung.com/opencsv">OpenCSV for parsing csv</a></li>
<li>  <a href="https://www.journaldev.com/12021/supercsv-csvbeanreader-cellprocessor-csvbeanwriter">SuperCSV for  parsing csv</a></li>
<li>  <a href="https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html">FLUX for reactive programming</a></li>
<li>  <a href="https://swagger.io/docs/">Swagger 2 for API documentation</a></li>
<li> Database backups </li> 
</ul>

**Database Backup:**
```Java
	@PreDestroy
	public void onDestroy() throws Exception {

		// create a backup in case of accident
		try {
			executeCommands();
		} catch (IOException | InterruptedException e1) {
			log.error("error in backup : '{0}'", e1);
		}

	}

	/**
	 * Execute commands using bash. execude a temporary script from java interface
	 *
	 * @throws IOException
	 *             Signals that an I/O exception has occurred.
	 * @throws InterruptedException
	 *             the interrupted exception
	 */
	public void executeCommands() throws IOException, InterruptedException {

		File tempScript = createTempScript();
		try {
			ProcessBuilder pb = new ProcessBuilder("bash", tempScript.toString());
			pb.inheritIO();
			Process process = pb.start();
			process.waitFor();
		} finally {
			tempScript.delete();
		}
	}

	/**
	 * Creates the temp script. the temp script is used to create a backup of
	 * database in case of accident; using pg_dump for postgresql database
	 *
	 * @return the file
	 * @throws IOException
	 *             Signals that an I/O exception has occurred.
	 */
	public File createTempScript() throws IOException {
		File tempScript = File.createTempFile("script", null);

		Writer streamWriter = new OutputStreamWriter(new FileOutputStream(tempScript));
		PrintWriter printWriter = new PrintWriter(streamWriter);
		SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");

		String dateString = format.format(new Date());
		printWriter.println("#!/bin/bash");
		printWriter.println(" PGPASSWORD=\"" + password + "\" pg_dump -U " + username + " "
				+ urlDatabase.substring(urlDatabase.lastIndexOf('/') + 1) + " > " + pathBackup + "/db" + dateString
				+ ".sql");

		printWriter.close();

		return tempScript;
	}

                  
```
<br><br>

**download from SFTP Server:**
```Java
	/**
	 * Download from server.
	 *
	 * @param host
	 *            the host of sftp server
	 * @param user
	 *            the user of sftp server
	 * @param password
	 *            the password of sftp server
	 * @param pathRemote
	 *            the path remote of the downloaded file
	 * @param pathLocal
	 *            the path local of the destination of the downloaded file
	 * @return the status of downloading from the SFTP/FTP server
	 */
	public String downloadFromServer(String host, String user, String password, int port, String pathRemote,
			String pathLocal) {
		JSch jsch = new JSch();
		Session session = null;
		try {
			session = jsch.getSession(user, host, port);
			session.setConfig("StrictHostKeyChecking", "no");
			session.setPassword(password);
			session.connect();

			Channel channel = session.openChannel("sftp");
			channel.connect();
			ChannelSftp sftpChannel = (ChannelSftp) channel;
			sftpChannel.get(pathRemote, pathLocal);
			log.info("download done successfully {}", "");
			sftpChannel.exit();

			session.disconnect();

			unzip(pathLocal + "lacotto_job_offer.zip", pathLocal);
			log.info("Unzip done successfully {}", "");
			return "Downloading and Unzipping done successfully  ";
		} catch (JSchException | SftpException e) {
			log.error(ERROR, e);
		}
		return "It seems there is a problem, please check the log for details";
	}

                  
```

<br><br>
**Unzip a file :**
```Java
	public void unzip(String zipFilePath, String destDir) {
		File dir = new File(destDir);
		// create output directory if it doesn't exist
		if (!dir.exists())
			dir.mkdirs();
		// buffer for read and write data to file
		byte[] buffer = new byte[1024];
		try (FileInputStream fis = new FileInputStream(zipFilePath); ZipInputStream zis = new ZipInputStream(fis);) {

			ZipEntry ze = zis.getNextEntry();

			while (ze != null) {
				String fileName = ze.getName();
				File newFile = new File(destDir + File.separator + fileName);
				log.info("Unzipping to {}", newFile.getAbsolutePath());
				// create directories for sub directories in zip
				new File(newFile.getParent()).mkdirs();
				try (FileOutputStream fos = new FileOutputStream(newFile)) {
					int len;
					while ((len = zis.read(buffer)) > 0) {
						fos.write(buffer, 0, len);
					}

					// close this ZipEntry
					zis.closeEntry();
					ze = zis.getNextEntry();
				}
			}
			// close last ZipEntry
			zis.closeEntry();

		} catch (IOException e) {
			log.error(ERROR, e);
		}

	}

                  
```

<br><br>

**Reactive CRUD with cache enabled :**
```Java
	**
 * The Class JobOtomeServiceImpl.
 */
@CacheConfig(cacheNames = { "jobOtomes" })
@Service
@Transactional
public class JobOtomeServiceImpl implements JobReactiveService {

	/** The job otome reactive repository. */
	@Autowired
	private JobOtomeReactiveRepository jobOtomeReactiveRepository;

	/** The transaction template. */
	@Autowired
	private TransactionTemplate transactionTemplate;

	/** The jdbc scheduler. */
	@Autowired
	@Qualifier("jdbcScheduler")
	private Scheduler jdbcScheduler;

	/*
	 * (non-Javadoc)
	 * 
	 * @see com.hrdatabank.otome.services.JobReactiveService#findById(long)
	 */
	@Override
	public Mono<Optional<JobOtome>> findById(long id) {
		return Mono.defer(() -> Mono.just(this.jobOtomeReactiveRepository.findById(id))).subscribeOn(jdbcScheduler);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see com.hrdatabank.otome.services.JobReactiveService#findAll()
	 */
	@Cacheable
	@Override
	public Flux<JobOtome> findAll() {
		return Flux.defer(() -> Flux.fromIterable(this.jobOtomeReactiveRepository.findAll()))
				.subscribeOn(jdbcScheduler);

	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.hrdatabank.otome.services.JobReactiveService#save(com.hrdatabank.otome.
	 * domain.JobOtome)
	 */
	@CachePut
	@Override
	public Mono<JobOtome> save(JobOtome job) {
		return Mono.fromCallable(() -> transactionTemplate.execute(status -> {
			return jobOtomeReactiveRepository.save(job);
		})).subscribeOn(jdbcScheduler);

	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.hrdatabank.otome.services.JobReactiveService#deleteById(java.lang.Long)
	 */
	@Override
	@CacheEvict(allEntries = true)
	public Mono<Void> deleteById(Long id) {
		jobOtomeReactiveRepository.deleteById(id);
		return Mono.empty();
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see com.hrdatabank.otome.services.JobReactiveService#findAllJobDTO()
	 */
	@Override
	@Cacheable
	public Flux<JobDto> findAllJobDTO() {
		try {
			Thread.sleep(3000L);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return Flux.defer(() -> Flux.fromIterable(this.jobOtomeReactiveRepository.getAllJobsByDto()))
				.subscribeOn(jdbcScheduler);

	}

}


                  
```

<br><br>


## Credits

This software uses code from several open source packages.

- [Journal Dev](https://www.journaldev.com)
- [Baeldung](www.baeldung.com)
- [tutorialspoint](https://www.tutorialspoint.com)

