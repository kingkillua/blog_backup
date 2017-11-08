---
title: javamail收件正文带图无法正常显示
date: 2015-03-07 21:30:30
tags: javamail
categories: 学
---
百度下其实就能知道问题所在：正文中的被cid链接取代
但是百度上代码却不能照搬，因为对邮件正文的解析已经跟以前不一样了
我这段代码也只能解析163和QQ邮箱的邮件，hotmail就不行，加上也没这要求就偷懒没继续完善了
但是总归就是断点一步步对应着调其实也没难度，博客刚开张写点东西


<!-- more -->
###邮件接收函数


``` java

private static final Logger log = Logger.getLogger(EmailUtil.class);
private static HashMap<String, String>	attrMap		= JFWebConfig.attrMap;
public static String mailhost =attrMap.get("mailhost");// "smtp.163.com";
public static String mailprot = attrMap.get("mailprot");//"smtp";	
public static String mailhost_re = attrMap.get("mailhost_re"); //"pop.163.com";
public static String mailprotocol_re = attrMap.get("mailprotocol_re");//"pop3";
public static String mailprot_re = attrMap.get("mailprot_re"); //"995";
//主邮件地址
public static String mailuser = attrMap.get("mailuser"); //"xxxxx@163.com";
public static String mailpwd = attrMap.get("mailpwd");//"xxxxx"
//正文图片map
private static HashMap<String,String> isMapImage=new HashMap();


	/**
	 * 邮件接收
	 * @param startnum 接收邮件开始序号,序号从1开始
	 * @param fjpath   附件存储路径（绝对路径）
	 * return 返回已接受邮件数
	 * */
	public static Integer receive(Integer startnum,String fjpath){
        Properties props = new Properties();
        props.put("mail.pop3.ssl.enable", "true");
        props.put("mail.pop3.host", mailhost_re);
        props.put("mail.pop3.port", mailprot_re);
        
        Session session = Session.getDefaultInstance(props);
        Store store = null;
        Folder folder = null;
        try {
            store = session.getStore(mailprotocol_re);
            store.connect(mailuser, mailpwd);
            folder = store.getFolder("INBOX");
            folder.open(Folder.READ_ONLY);
       
            int size = folder.getMessageCount();
            if(size>0 && startnum<size){
            	//Message message = folder.getMessage(size);
            	startnum = startnum +1;
                Message message[] = folder.getMessages(startnum, size);
                if(message!=null && message.length>0){
                	for(int i=0;i<message.length;i++){
                                //每次接受前清空这个正文图片map
                		isMapImage.clear();
                		//判断邮件文件夹是否打开
                		if(!message[i].getFolder().isOpen()){
                			message[i].getFolder().open(Folder.READ_ONLY);
                		}
                		//获取发件人信息
                		InternetAddress address[] = (InternetAddress[]) message[i].getFrom();   
                        String from = address[0].getAddress();   
                        if (from == null)   
                            from = "";   
                        String personal = address[0].getPersonal();   
                        if (personal == null)   
                            personal = "";   
                        String fromaddr = personal + "<" + from + ">";
                        //主题
                        String subject = message[i].getSubject();
                        //发送时间
                        Date date = message[i].getSentDate();
                        
                        //获得邮件的收件人，抄送，的地址和姓名
                        String address_to = getAddress(message[i],Message.RecipientType.TO);
                        String address_cc = getAddress(message[i],Message.RecipientType.CC);
                        
                        T_Bus_Mail_Rece rece = new T_Bus_Mail_Rece();
                        rece.set("subject", subject);
                        rece.set("sender", fromaddr);
                        rece.set("address_to", address_to);
                        rece.set("address_cc", address_cc);
                        rece.set("send_time", DateTimeUtil.formatDate(date, DateTimeUtil.Y_M_D_HMS_FORMAT));
                        rece.set("emseq", startnum+i);
                        rece.set("msgid", ((MimeMessage)message[i]).getMessageID());
                        //内容
                        
                        StringBuffer fjids = new StringBuffer();
                    	saveAttachMent((Part)message[i],fjpath+"/mail"+(startnum+i),fjids);
                    	rece.set("fjids", fjids.toString());
                    	
                        StringBuffer content = new StringBuffer();
                        getMailContent(content,(Part)message[i],fjpath+"/mail"+(startnum+i));
                        
                        rece.set("content", content.toString());
                        
                        ((com.sun.mail.pop3.POP3Message)message[i]).invalidate(true);//使缓存失效,解决OutOfMemory 异常
                        T_Bus_Mail_Rece.dao.insert(rece);
                        
                	} 
                }
            }
            return size;
          } catch (Exception e) {
            //e.printStackTrace();
            log.error(e.getMessage());
            return 0;
          } finally {
            try {
              if (folder != null) {
                folder.close(false);
              }
              if (store != null) {
                store.close();
              }
            } catch (Exception e) {
              //e.printStackTrace();
              log.error(e.getMessage());
            }
          }
       
	}

```


###获取整个邮件内容

``` java

        /**
	 * 获取邮件内容
	 * */
	public static void getMailContent(StringBuffer content, Part message,
			String mailfjPath) {
		try {
			String contenttype = message.getContentType();
			int nameindex = contenttype.indexOf("name");
			boolean conname = false;
			if (nameindex != -1) {
				conname = true;
			}
			boolean isContainTextAttach = message.getContentType().indexOf(
					"name") > 0;
			if (message.isMimeType("text/*") && !isContainTextAttach) {
				// 循环Map存在则替换 //需要将正文中cid置换成src路径
				String path = mailfjPath + "/";
				// 只取相对路径
				path = path.substring(path.lastIndexOf("WebRoot") + 7);
				String pictureContent = message.getContent().toString()
						.replace("cid:", path);
				Iterator ite = isMapImage.entrySet().iterator();
				while (ite.hasNext()) {
					Entry string = (Entry) ite.next();
					pictureContent = pictureContent.replace(string.getKey()
							.toString(), string.getValue().toString());
				}
				content.append(pictureContent);
			} else if (message.isMimeType("text/plain") && !conname) {
				content.append((String) message.getContent());
			} else if (message.isMimeType("text/html") && !conname) {
				content.append((String) message.getContent());
			} else if (message.isMimeType("multipart/*")) {
				Multipart multipart = (Multipart) message.getContent();
				int counts = multipart.getCount();
				boolean hasHtml = checkHasHtml(multipart);// 这里校验是否有text/html内容
				for (int i = 0; i < counts; i++) {
					Part temp = multipart.getBodyPart(i);
					if (temp.isMimeType("text/plain") && hasHtml) {
						// 有html格式的则不显示无格式文档的内容
					} else {
						getMailContent(content, multipart.getBodyPart(i),
								mailfjPath);
					}
				}
			} else if (message.isMimeType("message/rfc822")) {
				getMailContent(content, (Part) message.getContent(), mailfjPath);
			}
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}


```


###保存邮件中附件

```java

public static void saveAttachMent(Part part, String mailfjPath,
			StringBuffer fjids) throws Exception {
		String fileName = "";

		File mpath = new File(mailfjPath);
		if (!mpath.exists()) {
			mpath.mkdirs();
		}
		if (part.isMimeType("multipart/*")) {
			Multipart mp = (Multipart) part.getContent();
			for (int i = 0; i < mp.getCount(); i++) {
				BodyPart mpart = mp.getBodyPart(i);
				String disposition = mpart.getDisposition();
				if ((disposition != null)
						&& ((disposition.equals(Part.ATTACHMENT)) || (disposition
								.equals(Part.INLINE)))) {
					fileName = mpart.getFileName();
					if (fileName.toLowerCase().startsWith("=?gb")
							|| fileName.toLowerCase().startsWith("=?utf")) {
						fileName = fileName.replace("\r", "").replace("\n", "");
						fileName = MimeUtility.decodeText(fileName);
					}
					String fmailfjPath = mailfjPath + "/" + fileName;
					File file = new File(fmailfjPath);
					if (!file.exists()) {
						file.createNewFile();
					}
					BufferedOutputStream bos = null;
					BufferedInputStream bis = null;
					bos = new BufferedOutputStream(new FileOutputStream(file));
					bis = new BufferedInputStream(mpart.getInputStream());
					int c;
					while ((c = bis.read()) != -1) {
						bos.write(c);
						bos.flush();
					}
					bos.close();
					bis.close();

					File afile = new File(fmailfjPath);
					String m_type = fileName.substring(fileName
							.lastIndexOf('.') + 1);
					String m_size = FileUtils.getFileSize(afile.length());
					System.out.println("url:" + fmailfjPath + ",name:"
							+ fileName + ",m_type:" + m_type + ",m_size:"
							+ m_size);
					T_Attachment t = new T_Attachment();
					t.set("name", fileName);
					t.set("url", fmailfjPath);
					t.set("m_type", m_type);
					t.set("m_size", m_size);
					String id = T_Attachment.dao.insert(t);
					if (id != null && !"".equals(id)) {
						if (fjids.length() > 0) {
							fjids.append(",");
						}
						fjids.append(id);
					}
				} else if (mpart.isMimeType("multipart/*")) {
					saveAttachMent(mpart, mailfjPath, fjids);
				} else {
					fileName = mpart.getFileName();
					if ((fileName != null)
							&& (fileName.toLowerCase().startsWith("=?gb") || fileName
									.toLowerCase().startsWith("=?utf"))) {
						fileName = fileName.replace("\r", "").replace("\n", "");
						fileName = MimeUtility.decodeText(fileName);
						String fmailfjPath = mailfjPath + "/" + fileName;
						File file = new File(fmailfjPath);
						if (!file.exists()) {
							file.createNewFile();
						}
						BufferedOutputStream bos = null;
						BufferedInputStream bis = null;
						bos = new BufferedOutputStream(new FileOutputStream(
								file));
						bis = new BufferedInputStream(mpart.getInputStream());
						int c;
						while ((c = bis.read()) != -1) {
							bos.write(c);
							bos.flush();
						}
						bos.close();
						bis.close();
						File afile = new File(fmailfjPath);
						String m_type = fileName.substring(fileName
								.lastIndexOf('.') + 1);
						String m_size = FileUtils.getFileSize(afile.length());
						System.out.println("url:" + fmailfjPath + ",name:"
								+ fileName + ",m_type:" + m_type + ",m_size:"
								+ m_size);
						T_Attachment t = new T_Attachment();
						t.set("name", fileName);
						t.set("url", fmailfjPath);
						t.set("m_type", m_type);
						t.set("m_size", m_size);
						String id = T_Attachment.dao.insert(t);
						if (id != null && !"".equals(id)) {
							if (fjids.length() > 0) {
								fjids.append(",");
							}
							fjids.append(id);
						}
					} else {
						// 区别附件,正文图片此处保存
						String isCid = ""; 
						try {
							if (!mpart.getHeader("Content-ID")[0].equals("")) {
								isCid = mpart.getHeader("Content-ID")[0]
										.replace("<", "").replace(">", "");
							}
						} catch (Exception e) {
						}
						if (!isCid.equals("")) {
							// 保存正文的图片
							String fmailfjPath = mailfjPath + "/" + fileName
									+ ".jpg";
							File file = new File(fmailfjPath);
							if (!file.exists()) {
								file.createNewFile();
							}
							BufferedOutputStream bos = null;
							BufferedInputStream bis = null;
							bos = new BufferedOutputStream(
									new FileOutputStream(file));
							bis = new BufferedInputStream(
									mpart.getInputStream());
							int c;
							while ((c = bis.read()) != -1) {
								bos.write(c);
								bos.flush();
							}
							bos.close();
							bis.close();
							// 重命名正文中图片为cid
							renameFile(mailfjPath + fileName, mailfjPath
									+ isCid + ".jpg");
							// 插入正文图片键值对到正文图片map
							isMapImage.put(isCid, isCid + ".jpg");
						}
					}
				}
			}
		} else if (part.isMimeType("message/rfc822")) {
			saveAttachMent((Part) part.getContent(), mailfjPath, fjids);
		}
	} 
```

