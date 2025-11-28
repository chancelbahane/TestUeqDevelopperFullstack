# TestUeqDevelopperFullstack

/*pour résoudre le premier problème qui est de la réparation de l'infrastructure (Docker), il sied:
 - Vérification des logs de Dockers: Exécution docker-compose logs pour voir les logs de l'application et de la base de données
 - vérification de la configuration de la base de données: ici on veut s'assurer que l'application.yml est correct
 - verifcation du réseau docker*
 - vérification du volume docker*
 - on recréer lesconteneurs
 - verification des ports/

 //CODE PROPRIETES

spring.datasource.url=jdbc:postgresql://localhost:5432/NomDelaBD
spring.datasource.username=monuser
spring.datasource.password=monpassword
spring.jpa.hibernate.ddl-auto=update

//Code Java
package com.ueashop;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class UeaApplication {

  public static void main(String[] args) {
    SpringApplication.run(UeaApplication.class, args);
  }

//code Yml
version: '3'
services:
  db:
    image: postgres
    environment:
      - POSTGRES_USER=monuser
      - POSTGRES_PASSWORD=monpassword
      - POSTGRES_DB=mondb
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data

  app:
    build: .
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/mondb
      - SPRING_DATASOURCE_USERNAME=monuser
      - SPRING_DATASOURCE_PASSWORD=monpassword
    ports:
      - "8080:8080"
    depends_on:
      - db

volumes:
  db-data

//II.Pour le problème de l'optimisation de la base de donnéés:

/* -on doit activer leslogs hinernate
   - excecuter l'endpointGET/Categorie et analyser les logs Hibernate pour voir les requetes SQL générées
   - On echerche les requetes qui sont executées de manière repetée pour chargée les donnée de la catégorie
   */
//CODE Proprieties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type=TRACE
//Code Java

@Entity
public class Categorie {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String nom;
    @OneToMany(mappedBy = "categorie", fetch = FetchType.LAZY)
    private List<Produit> produits;

    // Getters et setters
}

public interface CategorieRepository extends JpaRepository<Categorie, Long> {
    @Query("SELECT c FROM Categorie c JOIN FETCH c.produits")
    List<Categorie> findAllCategoriesWithProduits();
}

@Service
public class CategorieService {
    @Autowired
    private CategorieRepository categorieRepository;

    public List<Categorie> getCategories() {
        return categorieRepository.findAllCategoriesWithProduits();
    }
}
III.IMPLANTATION DESFONCTIONNALITES
//modification de la methode placeOrder pour qu'elle verifie reelement le stock
//Codejava
@Service
public class OrderService {
    
    @Autowired
    private InventoryClient inventoryClient;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public void placeOrder(Order order) {
        // Vérifier le stock du produit
        Product product = order.getProduct();
        int quantity = order.getQuantity();
        boolean isInStock = inventoryClient.checkStock(product.getId(), quantity);
        
        if (!isInStock) {
            // Si le produit n'est pas en stock, annuler la commande et lancer une exception
            throw new ProductOutOfStockException("Le produit " + product.getName() + " n'est pas en stock");
        }
        
        // Si le produit est en stock, on crée la commande
        order.setStatus(OrderStatus.PLACED);
        orderRepository.save(order);
        
        // Réduire le stock du produit

/* voici comment on peut refactoriser la classe LegacyPaymentProcessor pour utiliser les standards Spring Boot
   */
   @Service
public class PaymentProcessor {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private Environment environment;
    
    @Value("${payment.processor.url}")
    private String paymentProcessorUrl;
    
    @Value("${payment.processor.username}")
    private String paymentProcessorUsername;
    
    @Value("${payment.processor.password}")
    private String paymentProcessorPassword;
    
    public void processPayment(Payment payment) {
        // Vérifier les informations de paiement
        if (!paymentRepository.existsById(payment.getId())) {
            throw new PaymentNotFoundException("Paiement non trouvé");
        }
        
        // Traiter le paiement
        String paymentProcessorResponse = sendPaymentRequest(payment);
        
        // Mettre à jour le statut du paiement
        payment.setStatus(PaymentStatus.PROCESSED);
        paymentRepository.save(payment);
    }
    
    private String sendPaymentRequest(Payment payment) {
        // Envoyer la requête de paiement au processeur de paiement
        RestTemplate restTemplate = new RestTemplate();
        HttpHeaders headers = new HttpHeaders();
        headers.setBasicAuth(paymentProcessorUsername, paymentProcessorPassword);
        HttpEntity<Payment> request = new HttpEntity<>(payment, headers);
        ResponseEntity<String> response = restTemplate.exchange(paymentProcessorUrl, HttpMethod.POST, request, String.class);
        return response.getBody();
    }
}

LE CANDIDAT BAHATI BAHANE CHANCE
