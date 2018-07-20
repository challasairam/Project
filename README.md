
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
