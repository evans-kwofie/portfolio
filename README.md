# My Portfolio Website

    <nav
    		aria-label="Main menu"
    		class="absolute -inset-x-4 top-14 hidden flex-col items-end gap-y-4 rounded-md bg-bgColor/[.85] py-4 text-link shadow backdrop-blur group-[.menu-open]:z-50 group-[.menu-open]:flex sm:static sm:z-auto sm:-ms-4 sm:mt-1 sm:flex sm:flex-row sm:items-center sm:divide-x sm:divide-dashed sm:divide-link sm:rounded-none sm:bg-transparent sm:py-0 sm:shadow-none sm:backdrop-blur-none"
    		id="navigation-menu"
    	>
    		{
    			menuLinks.map((link) => (
    				<a
    					aria-current={url.pathname === link.path ? "page" : false}
    					class="px-4 py-4 underline-offset-2 sm:py-0 sm:hover:underline"
    					data-astro-prefetch
    					href={link.path}
    				>
    					{link.title}
    				</a>
    			))
    		}
    	</nav>
# portfolio
