
@RequestMapping(value="fetchLoginDetails",method=RequestMethod.POST)
public ModelAndView getDetails(@ModelAttribute("user1") @Valid User user,BindingResult result,HttpServletRequest request)
{
	HttpSession session=request.getSession();
	ModelAndView view = new ModelAndView();
	
	System.out.println("Login Page");
	if(result.hasErrors())
	{
		view = new ModelAndView("login", "user1", user);
		ArrayList<String> roles = new ArrayList<String>();
		roles.add("User");
		roles.add("Employee");
		roles.add("Admin");
		view.addObject("roles", roles);
	}
	else
	{
		try {
			User user1=service.login(user);
			
			if(null!=user1)
			{
				session.setAttribute("username",user1.getUserName());
				if (user.getRole().equals("Admin")) {
					view=new ModelAndView("DisplayForAdmin","hotel",new Hotel());	
				} else {
					
					ArrayList<Hotel> hotels = service.getHotelList();
					
					if (!hotels.isEmpty()) {
						view=new ModelAndView("DisplayForUser","room",new RoomDetails());
						view.addObject("hotelList", hotels);
						
					} else {
						String msg = "No hotels are available for Booking!";
						view.addObject("msg", msg);
					}
					
				}
			}
			else
			{
				view = new ModelAndView("login", "user1", user);
				ArrayList<String> roles = new ArrayList<String>();
				roles.add("User");
				roles.add("Employee");
				roles.add("Admin");
				view.addObject("roles", roles);
				view.addObject("message", "Login Failed");
			}
			
		} catch (HotelException e) {
			
		}
		
	}
	return view;
	
}
-----------------------------------------------
controller
@RequestMapping("getRoomDetails")
	public ModelAndView getRoomDetails(
			@ModelAttribute("room") RoomDetails details) {
		ModelAndView view = new ModelAndView();
		try {
			boolean res = service.checkHotelId(details.getHotelId());
			if (res) {
				ArrayList<RoomDetails> list = service.getRoomDetails(details
						.getHotelId());
				System.out.println(list);
				if (!list.isEmpty()) {
					ArrayList<Hotel> hotels = service.getHotelList();
					view = new ModelAndView("DisplayForUser", "room",
							new RoomDetails());
					view.addObject("hotelList", hotels);
					view.addObject("roomDetails", list);

				} else {
					view = new ModelAndView("DisplayForUser", "room",
							new RoomDetails());
					String msg = "No rooms are available in this hotel for Booking!";
					view.addObject("rmsg", msg);
				}
			}
			else {
				view = new ModelAndView("DisplayForUser", "room",
						new RoomDetails());
				String msg = "No hotel exists with such hotel id!";
				view.addObject("hmsg", msg); 
			}

		} catch (HotelException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return view;

	}

	@RequestMapping("BookRoom")
	public ModelAndView bookARoom(@ModelAttribute("room") RoomDetails details) {

		BookingDetails bookingDetails = new BookingDetails();
		ModelAndView view = new ModelAndView("BookRoom", "book", bookingDetails);
		view.addObject("roomId", details.getRoomId());
		return view;
	}

	@RequestMapping("insertBooking")
	public ModelAndView insertBookingDetails(
			@ModelAttribute("book") BookingDetails bookingDetails,
			HttpServletRequest request) {
		HttpSession session = request.getSession(false);
		ModelAndView view = new ModelAndView();
		Object s = session.getAttribute("userid");
		bookingDetails.setUserId(Integer.parseInt(s.toString()));
		try {
			BookingDetails bookingDetails2 = service
					.insertBookingDetails(bookingDetails);
			view.addObject("id", bookingDetails2.getBookingId());
			view = new ModelAndView("DisplayForUser", "room", new RoomDetails());
		} catch (HotelException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return view;
	}

	@RequestMapping("bookingStatus")
	public ModelAndView viewBookingStatus(HttpServletRequest request) {
		HttpSession session = request.getSession(false);
		ModelAndView view = new ModelAndView();
		BookingDetails details = new BookingDetails();
		Object s = session.getAttribute("userid");
		details.setUserId(Integer.parseInt(s.toString()));
		try {
			boolean r = service.bookingStatus(details.getUserId());
			System.out.println(r);
			if (r) {
				view = new ModelAndView("DisplayForUser", "room",
						new RoomDetails());
				String statusmsg1 = "You have already booked!";
				view.addObject("statusmsg1", statusmsg1);
			} else {
				view = new ModelAndView("DisplayForUser", "room",
						new RoomDetails());
				String statusmsg2 = "Not yet booked!";
				view.addObject("statusmsg2", statusmsg2);
			}
		} catch (HotelException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		return view;
	}

	@RequestMapping("AddHotel")
	public ModelAndView getAddHotelPage() {
		ModelAndView view = new ModelAndView("AddHotel", "hotel", new Hotel());
		return view;

	}
	--------------------------------------------------------
	daoimpl
	public ArrayList<RoomDetails> getRoomDetails(Integer id)
			throws HotelException {

		Query query = entityManager.createNamedQuery("getRoomDetails");
		query.setParameter("id", id);
		ArrayList<RoomDetails> list = (ArrayList<RoomDetails>) query
				.getResultList();
		return list;
	}

	/*
	 * To insert booking details into database
	 */
	@Override
	public BookingDetails insertBookingDetails(BookingDetails bookingDetails)
			throws HotelException {
		DateTimeFormatter dateFormat = DateTimeFormatter
				.ofPattern("yyyy-MM-dd");
		LocalDate stDate = LocalDate.parse(bookingDetails.getBookedFromDate(),
				dateFormat);
		LocalDate endDate = LocalDate.parse(bookingDetails.getBookedToDate(),
				dateFormat);
		Period period = Period.between(stDate, endDate);
		Double diff = (double) period.getDays();
		RoomDetails details = entityManager.find(RoomDetails.class,
				bookingDetails.getRoomId());
		Double amount = details.getPerNightRate() * diff;
		bookingDetails.setAmount(amount);
		bookingDetails.setBookedFrom(Date.valueOf(stDate));
		bookingDetails.setBookedTo(Date.valueOf(endDate));
		System.out.println(bookingDetails);
		entityManager.persist(bookingDetails);
		RoomDetails roomDetails=entityManager.find(RoomDetails.class, bookingDetails.getRoomId());
		roomDetails.setAvailability("NA");
		entityManager.merge(roomDetails);
		entityManager.flush();
		return bookingDetails;
	}
	public boolean bookingStatus(int id) throws HotelException {
		boolean res= false;
		System.out.println("Id"+id);
		Query query = entityManager.createNamedQuery("bookingstatus");
		query.setParameter("uid", id);
		ArrayList<BookingDetails> list= (ArrayList<BookingDetails>) query.getResultList();
		System.out.println(list);
		if (!list.isEmpty())
			res = true;
		return res;
	}
	@Override
	public boolean checkHotelId(Integer hotelId) {
		// TODO Auto-generated method stub
		boolean res=false;
		Query query = entityManager.createNamedQuery("checkhotel");
		query.setParameter("id", hotelId);
		ArrayList<RoomDetails> list = (ArrayList<RoomDetails>) query.getResultList();
		if (!list.isEmpty())
			res = true;
		return res;
	}
	------------------------------------------------------
	displayuser.jsp
	<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
    <%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
    <%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Insert title here</title>
<style>
* {box-sizing: border-box}
body, html {
    height: 100%;
    margin: 0;
    font-family: Arial;
}
/* Style tab links */
.tablink {
    background-color: #555;
    color: white;
    float: left;
    border: none;
    outline: none;
    cursor: pointer;
    padding: 14px 16px;
    font-size: 17px;
    width: 25%;
}
.tablink:hover {
    background-color: #777;
   
}
.tabcontent {
    color: black;
    display: none;
    padding: 100px 20px;
    height: 100%;
}
#ViewListofHotels {background-color: white;}
#Bookaroom{background-color: white;}
#CheckBookingStatus {background-color: white;}
#Logout {background-color: white;}
</style>
<script src="./pages/script.js">
</script>

</head>
<body onload="f()">
<%
		HttpSession httpSession = request.getSession(false);
		if (null != httpSession) {
	%>
<%-- <input type="hidden" name="msg" value="${m }"> --%>
<button class="tablink" onclick="openPage('ViewListofHotels',this, 'black')">View List of Hotels</button>
<button class="tablink" onclick="openPage('Bookaroom', this, 'black')" id="defaultOpen">Book a room</button>
<button class="tablink" onclick="openPage('CheckBookingStatus', this, 'black')">Check Booking Status</button>
<button class="tablink" onclick="openPage('Logout', this, 'black')">Logout</button>



<div id="ViewListofHotels" class="tabcontent">
<%=session.getAttribute("username") %> 
 ${msg }
 
 <h1>Available Hotel details</h1>
 <table border="2">
              
            
                     <tr>
                     <th>Hotel Id</th>
                     <th>City</th>
                     <th>Hotel Name</th>
                     <th>Address</th>
                     <th>Description</th>
                     <th>Average Rate Per Night</th>
                     <th>Phone No.1</th>
                     <th>Phone No.2</th>
                     <th>Rating</th>
                     <th>Email Id</th>
                      <th>FAX</th>
                     
              </tr>
                <c:forEach items="${hotelList}" var="e">
                     <tr>
                           <td><c:out value="${e.hotelId}"></c:out></td>
                           <td><c:out value="${e.city}"></c:out></td>
                           <td><c:out value="${e.hotelName}"></c:out></td>
                           <td><c:out value="${e.address}"></c:out></td>
                           <td><c:out value="${e.description}"></c:out></td>
                           <td><c:out value="${e.avgRatePerNight}"></c:out></td>
                           <td><c:out value="${e.phoneNo1}"></c:out></td>
                           <td><c:out value="${e.phoneNo2}"></c:out></td>
               			    <td><c:out value="${e.rating}"></c:out></td>
                           <td><c:out value="${e.email}"></c:out></td>
                           <td><c:out value="${e.fax}"></c:out></td>
               			     
                     </tr>
                     
              </c:forEach>
              
              </table>
 
</div>

<div id="Bookaroom" class="tabcontent">
<%=session.getAttribute("username") %> 
${hmsg }
${rmsg }
  <h3>Booking</h3>
  <form:form action="getRoomDetails.obj" modelAttribute="room">
  Enter Hotel Id to retrieve Room Details
 <form:input path="hotelId"/>
 <form:errors path="hotelId"/>
 <input type="submit" name="search" value="Search">
 </form:form>
 <table border="2">
              
              
                     <tr>
                     <th>Hotel Id</th>
                     <th>Room Id</th>
                     <th>Room No</th>
                     <th>Room Type</th>
                     <th>Per Night rate</th>
                     <th>Availability</th>
                     
              </tr>
              <c:forEach items="${roomDetails}" var="e">
                     <tr><form:form action="BookRoom.obj" modelAttribute="room">
                           <td><form:hidden path="hotelId" value="${e.hotelId}"/><c:out value="${e.hotelId}"></c:out></td>
                           <td><form:hidden path="roomId" value="${e.roomId}"/><c:out value="${e.roomId}"></c:out></td>
                           <td><form:hidden path="roomNo" value="${e.roomNo}"/><c:out value="${e.roomNo}"></c:out></td>
                           <td><form:hidden path="roomType" value="${e.roomType}"/><c:out value="${e.roomType}"></c:out></td>
                           <td><form:hidden path="perNightRate" value="${e.perNightRate}"/><c:out value="${e.perNightRate}"></c:out></td>
                           <td><form:hidden path="availability" value="${e.availability}"/><c:out value="${e.availability}"></c:out></td>
               				<td>
							<input type="submit" value="Book Now" name="button">
							</form:form>      
                     </tr>
                     
              </c:forEach>
              
              </table>
 
</div>

<div id="CheckBookingStatus" class="tabcontent" onclick="status()">
  <a href="bookingStatus.obj">BookingStatus</a>
  ${statusmsg1 }
  ${statusmsg2 }
  
</div>

<div id="Logout" class="tabcontent">
 <!-- <h3>Logout</h3> -->
 <a href="index.obj">Logout</a>
</div>

<%} %>
</body>
</html>
----------------------------------------------
displayadmin.jsp
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
    <%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
    <%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Insert title here</title>
<style>
* {box-sizing: border-box}
/* Set height of body and the document to 100% */
body, html {
    height: 100%;
    margin: 0;
    font-family: Arial;
}
/* Style tab links */
.tablink {
    background-color: #555;
    color: white;
    float: left;
    border: none;
    outline: none;
    cursor: pointer;
    padding: 14px 16px;
    font-size: 17px;
    width: 25%;
}
.tablink:hover {
    background-color: #777;
}
/* Style the tab content (and add height:100% for full page content) */
.tabcontent {
    color: black;
    display: none;
    padding: 100px 20px;
    height: 100%;
}
#PerformHotelManagement {background-color: white;}
#PerformRoomManagement {background-color: white;}
#ViewReports {background-color: white;}
#Logout {background-color: white;}
</style>
<script src="./pages/script.js"></script>
</head>
<body>
<%
		HttpSession httpSession = request.getSession(false);
		if (null != httpSession) {
	%>
<button class="tablink" onclick="openPage('PerformHotelManagement', this, 'black')" id="defaultOpen">Perform Hotel Management</button>
<button class="tablink" onclick="openPage('PerformRoomManagement', this, 'black')">Perform Room Management</button>
<button class="tablink" onclick="openPage('ViewReports', this, 'black')">View Reports</button>
<button class="tablink" onclick="openPage('Logout', this, 'black')">Logout</button>

<div id="PerformHotelManagement" class="tabcontent">
 <center>
 <a href="AddHotel.obj">Add Hotel</a>
  <a href="DeleteHotel.obj">Delete Hotel</a>
   <a href="ModifyHotel.obj">Modify Hotel</a>

 </center>
</div>

<div id="PerformRoomManagement" class="tabcontent">
 <center>
  <a href="AddRoom.obj">Add Room</a>
  <a href="DeleteRoom.obj">Delete Room</a>
   <a href="ModifyRoom.obj">Modify Room</a>
 </center>
</div>

<div id="ViewReports" class="tabcontent">
  <center>
   <a href="ViewHotelList.obj">ViewHotelList</a>
  <a href="ViewBooking.obj">ViewBooking</a>
   <a href="ViewGuestList.obj">ViewGuestList</a>
   <a href="ViewSpecificBooking.obj">ViewSpecificBooking</a>
 </center>
</div>

<div id="Logout" class="tabcontent">
 <a href="index.obj">Logout</a>
</div>
<%} %>

</body>
</html>
----------------------------------------------------------
bookroom.jsp
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
    <%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Insert title here</title>
</head>
<body>
<%
		HttpSession httpSession = request.getSession(false);
		if (null != httpSession) {
	%>
<table>
<form:form action="insertBooking.obj" modelAttribute="book">
<%-- <tr>
<td>Enter room id: <form:input path="roomId"/>
<form:errors path="roomId"></form:errors></td>
</tr>
--%>

<form:hidden path="roomId" value="${roomId }"/>
<tr> 
<td>Enter Starting date in yyyy-MM-dd format: <form:input path="bookedFromDate"/>
<form:errors path="bookedFromDate"></form:errors></td>
</tr>

<tr>
<td>Enter Ending date in yyyy-MM-dd format: <form:input path="bookedToDate"/>
<form:errors path="bookedToDate"></form:errors></td>
</tr>

<tr>
<td>Enter no of adults: <form:input path="noOfAdults"/>
<form:errors path="noOfAdults"></form:errors></td>
</tr>

<tr>
<td>Enter no of Children: <form:input path="noOfChildren"/>
<form:errors path="noOfChildren"></form:errors></td>
</tr>
<tr>
<td>
<input type="submit" value="Book"></td>
</tr>
</form:form>
</table>
<%} %>
</body>
</html>
