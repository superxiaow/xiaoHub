package com.client;

import java.io.BufferedReader;
import java.io.DataOutputStream;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import org.dom4j.Document;
import org.dom4j.DocumentHelper;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;
import org.dom4j.io.XMLWriter;

import com.serotonin.modbus4j.ModbusMaster;
import com.serotonin.modbus4j.exception.ModbusInitException;
import com.serotonin.modbus4j.exception.ModbusTransportException;
import com.serotonin.modbus4j.msg.ModbusRequest;
import com.serotonin.modbus4j.msg.WriteCoilRequest;
import com.serotonin.modbus4j.msg.WriteRegisterRequest;

public class TestMyServlet {
	static ModbusMaster master;
	static int node;
	static boolean color;
	static boolean modsenseRunning = false;

	public TestMyServlet() {
		String version = "3.51";
		System.out.println("ModSense v" + version);
	}

	public static void main(String[] args) throws Exception {
		TestMyServlet clientRequest = new TestMyServlet();
		clientRequest.machine();
		Thread th = new Thread();
		do {
			System.out.print("\nChecking Process Status...");
			System.out.println(modsenseRunning);
			if (!modsenseRunning) {
				clientRequest.machine();
			}
			th.sleep(100000);
		} while (true);
	}

	public void machine() throws Exception{
		// TestMyServlet clientRequest = new TestMyServlet();

		modsenseRunning = true;
		/* Values to be set by properties file */
		int refreshTime = 10000;
		String dbAddress = "localhost";
		String dbUser = "root";
		String dbPassword = "b45yes";
		String dbTable = "modtest";
		String dbName = "";
		String connection = "";
		int offsetBegin = 0;
		int offsetLength = 100;

		node = 1;
		color = true;
		String serial = "";
		String pollOnly = "";
		Thread th = new Thread();	
		try {
			System.out.println("Loading modsense settings...");
			Properties props = new Properties();

			props.load(new FileInputStream("modsense.properties"));

			refreshTime = Integer.parseInt(props.getProperty("REFRESH"));
			connection = props.getProperty("CONNECTION");
			color = Boolean.parseBoolean(props.getProperty("COLOR"));
			node = Integer.parseInt(props.getProperty("UNITID"));
			dbName = props.getProperty("DBNAME");
			dbAddress = props.getProperty("DBADDRESS");
			dbTable = props.getProperty("DBTABLE");
			dbUser = props.getProperty("DBUSER");
			dbPassword = props.getProperty("DBPASSWORD");
			serial = props.getProperty("SERIAL");
			pollOnly = props.getProperty("POLLONLY");
			offsetBegin = Integer.parseInt(props.getProperty("OFFSETBEGIN"));
			offsetLength = Integer.parseInt(props.getProperty("OFFSETLENGTH"));

			System.out.println("Establishing connection to device...");
			FactoryBuilder fb = new FactoryBuilder(connection);
			master = fb.getMaster();
			master.setTimeout(1000);
			master.setRetries(0);
			master.init();
		} catch (FileNotFoundException e) {
			System.out.println("ERROR: Could not find properties file...");
        	modsenseRunning=false;
        	master.destroy();        	
        	e.printStackTrace();
		} catch (IOException e) {
			modsenseRunning=false;
        	master.destroy();   
			e.printStackTrace();
		} catch (ModbusInitException e) {
			System.out.println("ERROR: Could not establish connection to device...");
        	modsenseRunning=false;
        	master.destroy();        	
        	e.printStackTrace(); 
		}

		do {
			System.out.println("Beginning process...");
			List<String> queryAllData = new ArrayList<String>();
			List<String> updatedData = new ArrayList<String>();
			// 每隔几分钟，发出一次询问请求
			query_update(queryAllData, updatedData);
			// 调用一次query 返回所有值给server
			response_alldata(queryAllData, updatedData);
			th.sleep(refreshTime);
		} while (true);
	}

	public void query_update(List queryAllData, List updatedData) {

		Document document = DocumentHelper.createDocument();
		Element rootElement = document.addElement("request");
		/*
		 * Element typeElement = rootElement.addElement("type");
		 * typeElement.setText("query");
		 */
		// 以下部分从附件取得
		Element dbname = rootElement.addElement("dbname");
		dbname.setText("insite_test2");
		Element dbuser = rootElement.addElement("dbuser");
		dbuser.setText("root");
		Element dbpassword = rootElement.addElement("dbpassword");
		dbpassword.setText("root");
		Element serialElement = rootElement.addElement("serial");
		serialElement.setText("530152");

		HttpURLConnection http;
		System.out
				.println("---------------------------发送查询请求的plc---------------------------------");
		System.out.println(document.asXML());

		try {
			URL urls = new URL(
					"http://xwangw7lp.fulton.local:8080/customplatform/test");
			http = (HttpURLConnection) urls.openConnection();
			http.setDoOutput(true);
			http.setDoInput(true);
			http.setRequestMethod("POST");
			DataOutputStream out = new DataOutputStream(http.getOutputStream());
			XMLWriter writer = new XMLWriter(out);
			writer.write(document);
			writer.close();
			out.flush();

			System.out
					.println("-----------------------接收服务器反馈的更新数据和所有数据地址----------------------------");
			InputStream resp_in = http.getInputStream();
			SAXReader saxReader = new SAXReader();
			InputStreamReader resp_strInStream = new InputStreamReader(resp_in,
					"UTF-8");
			Document resp_document = saxReader.read(resp_strInStream);
			Element resp_root = resp_document.getRootElement();
			Iterator resp_lv = resp_root.elementIterator("update-item");
			Element resp_el = null;
			while (resp_lv.hasNext()) {// 循环获得需要更新的address和value:
				resp_el = (Element) resp_lv.next();
				System.out.println(resp_el.elementText("address") + ":"
						+ resp_el.elementText("value"));
				// if mb if mi if ml, 不同类型的address 在这里分类
				if (resp_el.elementText("address").length() == 5) {
					if (writeToAddress(resp_el.elementText("address"),
							resp_el.elementText("value"))) {
						updatedData.add(resp_el.elementText("address")); // 如果修改成功，把数据放入修改成功的数组里面
					}
				} else if (resp_el.elementText("address").length() == 4) {
					if (writeHoldingToAddress(resp_el.elementText("address"),
							resp_el.elementText("value"))) {
						updatedData.add(resp_el.elementText("address"));
					}
				}

			}
			Iterator resp_lv2 = resp_root.elementIterator("query-item");
			Element resp_el2 = null;
			while (resp_lv2.hasNext()) {
				resp_el2 = (Element) resp_lv2.next();
				queryAllData.add(resp_el2.elementText("address"));
			}
		} catch (Exception ex) {
			ex.printStackTrace();
		} 
	}

	public void response_alldata(List queryAllData, List updatedData) {
		Document document = DocumentHelper.createDocument();
		Element rootElement = document.addElement("feedback");
		Element dbname = rootElement.addElement("dbname");
		dbname.setText("insite_test2");
		Element dbuser = rootElement.addElement("dbuser");
		dbuser.setText("root");
		Element dbpassword = rootElement.addElement("dbpassword");
		dbpassword.setText("root");
		Element serialElement = rootElement.addElement("serial");
		serialElement.setText("530152");

		Map resp_alldata = new HashMap();
		for (int i = 0; i < queryAllData.size(); i++) {
			// String value=getAddressValue(queryAllData[k]);
			String value = "x";// 从Plc读出来的
			resp_alldata.put(queryAllData.get(i), value);
		}

		Iterator iter = resp_alldata.entrySet().iterator();
		while (iter.hasNext()) {
			Map.Entry entry = (Map.Entry) iter.next();
			String address = (String) entry.getKey();
			String val = (String) entry.getValue();
			Element element = rootElement.addElement("response-item");
			Element addressElement = element.addElement("address");
			Element valueElement = element.addElement("value");
			addressElement.setText(address);
			valueElement.setText(val);
		}
		for (int i = 0; i < updatedData.size(); i++) {
			Element element = rootElement.addElement("updated-item");
			Element addressElement = element.addElement("address");
			addressElement.setText((String) updatedData.get(i));
		}
		System.out
				.println("------------------发送所有plc数据的xml-------------------");
		System.out.println(document.asXML());
		HttpURLConnection http;

		try {
			URL urls = new URL(
					"http://xwangw7lp.fulton.local:8080/customplatform/test");
			http = (HttpURLConnection) urls.openConnection();
			http.setDoOutput(true);
			http.setDoInput(true);
			http.setRequestMethod("POST");
			DataOutputStream out = new DataOutputStream(http.getOutputStream());
			XMLWriter writer = new XMLWriter(out);
			writer.write(document);
			writer.close();
			out.flush();

			String resp = "";
			java.io.BufferedReader breader = new BufferedReader(
					new InputStreamReader(http.getInputStream(), "UTF-8"));
			String str = breader.readLine();
			while (str != null) {
				resp += str;
				str = breader.readLine();
			}
			System.out.println("resp=" + resp);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

	public static boolean writeToAddress(String address, String value) { // KEY:10001
																			// VALUE:1
		// String addressType = address.substring(0,2);//MB
		int addressOffset = Integer.parseInt(address);// 01 China: 00001

		boolean boolValue = Boolean.parseBoolean(value);
		return writeMI(addressOffset, boolValue);

	}

	public static boolean writeMI(int offset, boolean value) {
		try {
			ModbusRequest req = new WriteCoilRequest(node, offset, value);
			master.send(req);
			return true;
		} catch (ModbusTransportException ex) {
			return false;
		}
	}

	public static boolean writeHoldingToAddress(String address, String value) {
		int addressOffset = Integer.parseInt(address);// 40002
		int intValue = Integer.parseInt(value);
		return writeHolding(addressOffset, intValue);
	}

	public static boolean writeHolding(int offset, int value) {

		try {
			ModbusRequest req = new WriteRegisterRequest(node, offset, value);
			master.send(req);
			return true;
		} catch (ModbusTransportException ex) {
			return false;
		}
	}

}
