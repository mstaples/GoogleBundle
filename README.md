Introduction
============
This Bundle is an old version of the bitgandtter/google-bundle.
I am maintaining this independently as this version is used in my current project.
Note: this branch is compatible with releases of Symfony2 >= v2.3

Installation
============

  1. Add this bundle and the Google PHP SDK to your ``vendor/`` dir:
      * Using the vendors script.

        Add the following lines in your ``deps`` file::

            {
            "require": {
                "bitgandtter/google-bundle": "0.1"
            	}
            }

  2. Run the composer to download the bundle
            
            $ composer update
          
  
  3. Add this bundle to your application's kernel:

          // app/ApplicationKernel.php
          public function registerBundles()
          {
              return array(
                  // ...
                  new BIT\GoogleBundle\BITGoogleBundle(),
                  // ...
              );
          }
          
  4. Add the following routes to your application and point them at actual controller actions
          
          #application/config/routing.yml
          _security_check:
              pattern:  /login_check
          _security_logout:
              pattern:  /logout

          #application/config/routing.xml
          <route id="_security_check" pattern="/login_check" />
          <route id="_security_logout" pattern="/logout" />     

  5. Configure the `google` service in your config:

          # application/config/config.yml
          bit_google:
      	    app_name: appName
      	    client_id: 123456789
      	    client_secret: s3cr3t
      	    state: auth
      	    access_type: online
      	    scopes: [userinfo.email, userinfo.profile]
      	    approval_prompt: auto
      	    callback_url: http://yourdomain.com/login_check?google=true
      	    
  NOTE: this extra parameter in the callback_url is mandatory needed to locate the google firewall

  6. Add this configuration if you want to use the `security component`:

          # application/config/config.yml
          security:
              firewalls:
                  public:
                      # since anonymous is allowed users will not be forced to login
                      pattern:   ^/.*
		      bit_google:
			        provider: google

              access_control:
                  - { path: ^/secured/.*, role: [IS_AUTHENTICATED_FULLY] } # This is the route secured with bit_google
                  - { path: ^/.*, role: [IS_AUTHENTICATED_ANONYMOUSLY] }

     You have to add `/secured/` in your routing for this to work. An example would be...
     
              _google_secured:
                  pattern: /secured/
                  defaults: { _controller: AcmeDemoBundle:Welcome:index }

  7. Optionally define a custom user provider class and use it as the provider or define path for login

          # application/config/config.yml
          security:
              providers:
                  # choose the provider name freely
		              google:
		                id: google.user # see "Example Customer User Provider using the BIT\UserBundle" chapter further down

              firewalls:
                  public:
                      pattern:   ^/.*
                      bit_google:
			            provider: google
                      anonymous: true
                      logout: true 

  8. Optionally use access control to secure specific URLs


          # application/config/config.yml
          security:
              # ...
              
              access_control:
                  - { path: ^/google/,           role: [ROLE_GOOGLE] }
                  - { path: ^/.*,                role: [IS_AUTHENTICATED_ANONYMOUSLY] }

Include the login button in your templates
------------------------------------------

Just add the following code in one of your templates:

    {{ google_login_button() }}
    
Or if you want to use:

	{% autoescape false %}{{ google_login_url() }}{% endautoescape %}
	
This will get you the login url

Example Customer User Provider using the FOS\UserBundle
-------------------------------------------------------

This requires adding a service for the custom user provider which is then set
to the provider id in the "provider" section in the config.yml:

    services:
        google.user:
	  class: class: Acme\MyBundle\Security\User\Provider\googleProvider
	  arguments:
	      google: @bit_google.api
	      userManager: @bit_user.user_manager
	      validator: @validator
	      em: @doctrine.orm.entity_manager

    <?php
	Acme\MyBundle\Security\User\Provider;
	use Symfony\Component\Security\Core\Exception\UsernameNotFoundException;
	use Symfony\Component\Security\Core\Exception\UnsupportedUserException;
	use Symfony\Component\Security\Core\User\UserProviderInterface;
	use Symfony\Component\Security\Core\User\UserInterface;

	class GoogleProvider implements UserProviderInterface
	{
	  /**
	   * @var \GoogleApi
	   */
	  protected $googleApi;
	  protected $userManager;
	  protected $validator;
	  protected $em;

	  public function __construct( $googleApi, $userManager, $validator, $em )
	  {
	    $this->googleApi = $googleApi;
	    $this->userManager = $userManager;
	    $this->validator = $validator;
	    $this->em = $em;
	  }

	  public function supportsClass( $class )
	  {
	    return $this->userManager->supportsClass( $class );
	  }

	  public function findUserByGIdOrEmail( $gId, $email = null )
	  {
	    $user = $this->userManager->findUserByUsernameOrEmail( $email );
	    if ( !$user )
	      $user = $this->userManager->findUserBy( array( 'googleID' => $gId ) );
	    return $user;
	  }

	  public function loadUserByUsername( $username )
	  {
	    try
	    {
	      $gData = $this->googleApi->getOAuth( )->userinfo->get( );
	    }
	    catch ( \Exception $e )
	    {
	      $gData = null;
	    }

	    if ( !empty( $gData ) )
	    {

	      $email = $gData["email"];
	      $user = $this->findUserByGIdOrEmail( $username, isset( $email ) ? $email : null );

	      if ( empty( $user ) )
	      {
      		$user = $this->userManager->createUser( );
      		$user->setEnabled( true );
      		$user->setPassword( '' );
      		$user->setSalt( '' );
	      }

	      $id = $gData->getId();

	      if ( isset( $id ) )
	      {
		      $user->setGoogleID( $id );
	      }

	      $name = $gData["full_name"];

	      if ( isset( $name ) )
	      {
      		$nameAndLastNames = explode( " ", $name );
      		if ( count( $nameAndLastNames ) > 1 )
      		{
      		  $user->setFirstname( $nameAndLastNames[0] );
      		  $user->setLastname( $nameAndLastNames[1] );
      		  $user->setLastname2( ( count( $nameAndLastNames ) > 2 ) ? $nameAndLastNames[2] : "" );
      		}
      		else
      		{
      		  $user->setFirstname( $nameAndLastNames[0] );
      		  $user->setLastname( "" );
      		  $user->setLastname2( "" );
      		}
	      }

	      if ( isset( $email ) )
	      {
      		$user->setEmail( $email );
      		$user->setUsername( $email );
	      }
	      else
	      {
      		$user->setEmail( $id . "@google.com" );
      		$user->setUsername( $id . "@google.com" );
	      }

	      if ( count( $this->validator->validate( $user, 'Google' ) ) )
	      {
		// TODO: the user was found obviously, but doesnt match our expectations, do something smart
		throw new UsernameNotFoundException( 'The google user could not be stored');
	      }
	      $this->userManager->updateUser( $user );
	    }

	    if ( empty( $user ) )
	    {
	      throw new UsernameNotFoundException( 'The user is not authenticated on google');
	    }

	    return $user;
	  }

	  public function refreshUser( UserInterface $user )
	  {
	    if ( !$this->supportsClass( get_class( $user ) ) || !$user->getGoogleId( ) )
	    {
	      throw new UnsupportedUserException( sprintf( 'Instances of "%s" are not supported.', get_class( $user ) ));
	    }

	    return $this->loadUserByUsername( $user->getGoogleId( ) );
	  }
	}


Finally one also needs to add a getGoogleId() method to the User model.
The following example also adds "firstname" and "lastname" properties:

    <?php

	Acme\MyBundle\Entity;
	use Doctrine\Common\Collections\ArrayCollection;
	use FOS\UserBundle\Entity\User as BaseUser;
	use Doctrine\ORM\Mapping as ORM;

	/**
	 * @ORM\Entity
	 * @ORM\Table(name="system_user")
	 */
	class User extends BaseUser
	{
	  /**
	   * @ORM\Id
	   * @ORM\Column(type="integer")
	   * @ORM\generatedValue(strategy="AUTO")
	   */
	  protected $id;

	  /**
	   * @ORM\Column(type="string", length=40, nullable=true)
	   */
	  protected $googleID;

	  /**
	   * @ORM\Column(type="string", length=100, nullable=true)
	   */
	  protected $firstname;

	  /**
	   * @ORM\Column(type="string", length=100, nullable=true)
	   */
	  protected $lastname;

	  /**
	   * @ORM\Column(type="string", length=100, nullable=true)
	   */
	  protected $lastname2;

	  public function __construct( )
	  {
	    parent::__construct( );
	  }

	  public function getId( )
	  {
	    return $this->id;
	  }

	  public function getFirstName( )
	  {
	    return $this->firstname;
	  }

	  public function setFirstName( $firstname )
	  {
	    $this->firstname = $firstname;
	  }

	  public function getLastName( )
	  {
	    return $this->lastname;
	  }

	  public function setLastName( $lastname )
	  {
	    $this->lastname = $lastname;
	  }

	  public function getLastName2( )
	  {
	    return $this->lastname2;
	  }

	  public function setLastName2( $lastname2 )
	  {
	    $this->lastname2 = $lastname2;
	  }

	  public function getFullName( )
	  {
	    $fullName = ( $this->getFirstName( ) ) ? $this->getFirstName( ) . ' ' : '';
	    $fullName .= ( $this->getLastName( ) ) ? $this->getLastName( ) . ' ' : '';
	    $fullName .= ( $this->getLastName2( ) ) ? $this->getLastName2( ) . ' ' : '';
	    return $fullName;
	  }

	  public function setGoogleID( $googleID )
	  {
	    $this->googleID = $googleID;
	  }

	  public function getGoogleID( )
	  {
	    return $this->googleID;
	  }


	  public function setSalt( $salt )
	  {
	    $this->salt = $salt;
	  }
	}

