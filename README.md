# Spring and Multiple Entities

## Learning Goals

- Review one-to-many relationships and many-to-many relationships.
- Learn how we can apply these relationships in Spring Boot.

## Introduction

In the last module, we learned about the different relationships database tables
can have with one another. Let's review the many-to-many relationship and the
one-to-many relationships and then see how we can query these relationships in
Spring Boot.

## JPA Relationship Review

There are three kinds of relationships we talk about when describing database
tables:

- One-to-One: This is when one record in a table will relate to exactly one
  other record in another table.
  - Example: A country has exactly one capital city and each capital belongs
    to exactly one country.
- One-to-Many: This is when one record in a table can relate to multiple
  records in another table.
  - Example: An employee works for only one department but a department can
    have many employees.
- Many-to-Many: This is when multiple records in a table can relate to multiple
  other records in another table.
  - Example: A film may have many actors and an actor may be in many films.

The more common relationships we see are the one-to-many relationships and the
many-to-many relationships. Let's see how to set up these relationships again
in Spring Boot. Consider the following ER diagram:

![football-ER-diagram](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/football-team-and-football-player-schema.png)

Let's modify our schema now in our sports database to add this new table and
this many-to-many relationship:

```postgresql
DROP TABLE IF EXISTS football_player;
DROP TABLE IF EXISTS football_team;

CREATE TABLE football_team (
 id INTEGER PRIMARY KEY,
 team_name TEXT NOT NULL,
 wins Integer,
 losses INTEGER,
 current_super_bowl_champion BOOLEAN
);

CREATE TABLE football_player (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  position TEXT,
  team_id INTEGER NOT NULL,
  CONSTRAINT team_id FOREIGN KEY (team_id) REFERENCES football_team(id)
    ON DELETE CASCADE
);
```

In our spring-data-demo project, add an entity class called `FootballPlayer` to
the `entity` package with the following code:

```java
package com.example.springdatademo.entity;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Getter
@Setter
@NoArgsConstructor
@Table(name = "football_player")
public class FootballPlayer {

    @Id
    @GeneratedValue
    private int id;

    private String name;

    private String position;
    
    private FootballTeam footballTeam;
}
```

Now we have to relate this entity to the `FootballTeam` entity. Our ER diagram
and schema state that one football team can have many football players, but a
football player can only belong to one football team.

Since the football player is on the "many" side of this relationship, we'll
add the annotation `@ManyToOne`, just as we saw in the last module. This will
also add the import statement `import javax.persistence.ManyToOne;` We will also
add the annotation `@JoinColumn` to specify that this field would be joined by
the `team_id` attribute.

```java
@ManyToOne
@JoinColumn(name = "team_id")
private FootballTeam footballTeam;
```

And just as we saw in the last module, we'll have to make some modifications to
our `FootballTeam` entity now to add this relationship. In the `FootballTeam`
class, go ahead and add the following field:

```java
@OneToMany
private List<FootballPlayer> players = new ArrayList<>();
```

## One-to-Many Relationships in Spring Boot

Cool! We have set up our entity classes! This all should have been a review
from the last module. Now the rest of our classes, starting with the DTOs.

Say when we retrieve a football player, we want to know the team name they play
on; but, when we add a new football player to our database we want to still
inject the football team's ID. Let's create a `FootballPlayerDTO` class in the
`dto` package to reflect these requirements:

```java
package com.example.springdatademo.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.Data;

import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.NotNull;

@Data
public class FootballPlayerDTO {
    
    @NotNull
    @NotEmpty
    private String name;
    
    private String position;
    
    @JsonProperty(value = "team_id", access = JsonProperty.Access.WRITE_ONLY)
    private int teamId;

    @JsonProperty(value = "team_name", access = JsonProperty.Access.READ_ONLY)
    private String teamName;
}
```

As we saw in the "DTOs Revisited" lesson, we are only going to serialize the
`teamId` field, meaning we'll take it in but will not display it when returning
the DTO to the client, and we are only going to deserialize the `teamName`
field.

We're also going to have to add another repository interface to access the
`football_player` table. In the `repository` package, let's adjust our
`FootballRepository` interface name. Change `FootballRepository` to
`TeamRepository` by right-clicking the interface's name in the project
structure view in IntelliJ, then going down to "Refactor" and then "Rename":

![rename-football-repository](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/rename-interface.png)

A dialog window will then appear. Type in "TeamRepository" to say we want to
rename this file from `FootballRepository` to `TeamRepository`:

![rename-football-repository-dialog](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/rename-dialog.png)

If a "Rename Variables" window pops up, select all that are there to rename the
variables throughout the project. A refactor window may pop up in the bottom of
IntelliJ where the log and terminal usually appear. When that window pops up,
click "Do Refactor":

![rename-vairable-dialog](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/rename-variables.png)

Now go through the project and classes and make sure that the
`FootballRepository` interface has been renamed to `TeamRepository`.

Let's add a new repository to interface with the `football_player` table now.
In the `repository` package, create a new interface called `PlayerRepository`:

```java
package com.example.springdatademo.repository;

import com.example.springdatademo.entity.FootballPlayer;
import org.springframework.data.repository.CrudRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface PlayerRepository extends CrudRepository<FootballPlayer, Integer> {
    
    Optional<FootballPlayer> findFootballPlayerByName(String name);
}
```

We now have two repository interfaces: one to interface with the `football_team`
database table and one to interface with the `football_player`.

Now let's implement some methods in our controller and service classes. Consider
the following changes to the `FootballController` and `FootballService`:

```java
// FootballService.java

package com.example.springdatademo.service;

import com.example.springdatademo.dto.FootballPlayerDTO;
import com.example.springdatademo.dto.FootballTeamDTO;
import com.example.springdatademo.dto.FootballTeamNoChampionDTO;
import com.example.springdatademo.dto.FootballTeamWithChampionDTO;
import com.example.springdatademo.entity.FootballPlayer;
import com.example.springdatademo.entity.FootballTeam;
import com.example.springdatademo.repository.PlayerRepository;
import com.example.springdatademo.repository.TeamRepository;
import org.modelmapper.ModelMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

@Service
public class FootballService {

  private final ModelMapper modelMapper;
  private final TeamRepository teamRepository;
  private final PlayerRepository playerRepository;

  @Autowired
  public FootballService(ModelMapper modelMapper, TeamRepository teamRepository, PlayerRepository playerRepository) {
    this.modelMapper = modelMapper;
    this.teamRepository = teamRepository;
    this.playerRepository = playerRepository;
  }

  public String addFootballTeam(FootballTeamWithChampionDTO footballTeam) {
    FootballTeam footballTeamEntity = modelMapper.map(footballTeam, FootballTeam.class);
    teamRepository.save(footballTeamEntity);
    return String.format("%s has been added!", footballTeam.getTeamName());
  }

  public FootballTeamWithChampionDTO getFootballTeam(String teamName) {
    Optional<FootballTeam> optionalFootballTeam = teamRepository.findFootballTeamByTeamName(teamName);
    FootballTeam footballTeamEntity = optionalFootballTeam.orElseThrow();
    return modelMapper.map(footballTeamEntity, FootballTeamWithChampionDTO.class);
  }

  public List<FootballTeamNoChampionDTO> getAllFootballTeams() {
    List<FootballTeamNoChampionDTO> footballTeamDTOS = new ArrayList<>();
    Iterable<FootballTeam> footballTeams = teamRepository.findAll();
    for (FootballTeam footballTeam : footballTeams) {
      FootballTeamNoChampionDTO footballTeamDTO = modelMapper.map(footballTeam, FootballTeamNoChampionDTO.class);
      footballTeamDTOS.add(footballTeamDTO);
    }

    return footballTeamDTOS;
  }

  public String updateFootballTeam(Integer id, FootballTeamDTO footballTeamDTO) {
    Optional<FootballTeam> optionalFootballTeam = teamRepository.findById(id);
    if (optionalFootballTeam.isPresent()) {
      FootballTeam footballTeamEntity = optionalFootballTeam.get();
      modelMapper.map(footballTeamDTO, footballTeamEntity);
      teamRepository.save(footballTeamEntity);
      return String.format("Team with ID %d has been updated", id);
    } else {
      return String.format("Team with ID %d was not updated; ID may not exist.", id);
    }
  }

  public String deleteFootballTeam(Integer id) {
    teamRepository.deleteById(id);
    return String.format("Team with ID %d was deleted", id);
  }

  /**
   * Add a new football player to the sports database
   * @param footballPlayer : FootballPlayerDTO - football player to be added
   * @return String - message saying the football player was added
   */
  public String addFootballPlayer(FootballPlayerDTO footballPlayer) {
    FootballPlayer footballPlayerEntity = modelMapper.map(footballPlayer, FootballPlayer.class);
    playerRepository.save(footballPlayerEntity);
    return String.format("%s has been added!", footballPlayer.getName());
  }

  /**
   * Get a football player from the sports database based off the player's name
   * @param playerName : String - football player's name to query and retrieve
   * @return FootballPlayerDTO - data about the football player with the corresponding name
   */
  public FootballPlayerDTO getFootballPlayer(String playerName) {
    Optional<FootballPlayer> optionalFootballPlayer = playerRepository.findFootballPlayerByName(playerName);
    FootballPlayer footballPlayer = optionalFootballPlayer.orElseThrow();
    return modelMapper.map(footballPlayer, FootballPlayerDTO.class);
  }
}
```

```java
package com.example.springdatademo.controller;

import com.example.springdatademo.dto.FootballPlayerDTO;
import com.example.springdatademo.dto.FootballTeamDTO;
import com.example.springdatademo.dto.FootballTeamNoChampionDTO;
import com.example.springdatademo.dto.FootballTeamWithChampionDTO;
import com.example.springdatademo.service.FootballService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.List;

@RestController
public class FootballController {

    private final FootballService footballService;

    @Autowired
    public FootballController(FootballService footballService) {
        this.footballService = footballService;
    }

    @PostMapping("/football-team")
    public ResponseEntity<String> addFootballTeam(@Valid @RequestBody FootballTeamWithChampionDTO footballTeam) {
        String status = footballService.addFootballTeam(footballTeam);
        return ResponseEntity.ok(status);
    }

    /**
     * Get a football team by the team name
     * @param teamName : String - name of the football team of interest
     * @return FootballTeamDTO
     */
    @GetMapping("/football-team/{teamName}")
    public FootballTeamWithChampionDTO getFootballTeam(@PathVariable String teamName) {
        return footballService.getFootballTeam(teamName);
    }

    /**
     * Get all the football teams in the data source
     * @return List<FootballTeamDTO>
     */
    @GetMapping("/football-teams")
    public List<FootballTeamNoChampionDTO> getFootballTeams() {
        return footballService.getAllFootballTeams();
    }

    @PutMapping("/football-team/{footballId}")
    public String updateFootballTeam(@PathVariable Integer footballId, @RequestBody FootballTeamDTO footballTeam) {
        return footballService.updateFootballTeam(footballId, footballTeam);
    }

    @DeleteMapping("football-team/{footballId}")
    public String deleteFootballTeam(@PathVariable Integer footballId) {
        return footballService.deleteFootballTeam(footballId);
    }

  /**
   * Persist a football player to the database
   * @param footballPlayer : FootballPlayerDTO - the data of the football player to be added
   * @return String
   */
  @PostMapping("/football-player")
    public ResponseEntity<String> addFootballPlayer(@Valid @RequestBody FootballPlayerDTO footballPlayer) {
        String status = footballService.addFootballPlayer(footballPlayer);
        return ResponseEntity.ok(status);
    }

  /**
   * Get a football player by its name
   * @param playerName : String - name of the football player to be retrieved
   * @return FootballPlayerDTO
   */
  @GetMapping("/football-player/{playerName}")
    public FootballPlayerDTO getFootballPlayer(@PathVariable String playerName) {
        return footballService.getFootballPlayer(playerName);
    }
}
```

Note: In this example, we just simply added the football player methods to the
existing `FootballController` and `FootballService` classes. While this is one
way we could do this, we could also have created separate controller and service
classes: one to only serve the `FootballTeam` entity and one to only serve the
`FootballPlayer` entity. Note that this is a design choice; however, with more
entities, it would be best to separate them out into separate classes. You will
be doing this with your project coming up.

In both classes, we added two new methods: one to POST a football player to the
database and one to GET a football player from the database based on the
player's name.

Let's run the application now to test our new code. Open up Postman and add the
following JSON:

```json
{
  "name":"Dak-Prescott",
  "position": "Quarterback",
  "team_id": 1
}
```

Choose "POST" for the request type and type in
"http://localhost:8080/football-player" for the request URL. Click "Send"
to persist the data:

![add-football-player](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/postman-add-football-player.png)

We can double check in pgAdmin4 that the football player was added by performing
a select all query as such:

![select-all-football-player](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/pgadmin-select-all-3.png)

Now let's test the GET request. In Postman, select the GET request type and
enter "http://localhost:8080/football-player/Dak-Prescott" for the request URL.
Click "Send" to send the request:

![get-football-player](https://curriculum-content.s3.amazonaws.com/spring-mod-1/multiple-entities/postman-get-football-player.png)


## Many-to-Many Relationships in Spring Boot

If we were to implement a many-to-many relationship, we would utilize the
`@ManyToMany` annotations we used in the "Many-toMany Relationship" lesson we
saw in the last module - just as we used the `@ManyToOne` and `@OneToMany`
annotations here with the football example.

Of course, with a many-to-many relationship, the rule goes as follows:

> If the relationship is many-to-many, create a new table with the primary keys
> of each entity's table. For example, a book has many authors and an author may
> write many books. In this case, we would create a new table called
> `book_author` with a composite primary key (book_id, author_id), with each
> id being a foreign key to the corresponding table.
 
This would then create two one-to-many relationships which could then also be
implemented as we saw above.

## Conclusion

As we can see, not much has been changed here when we add multiple
relationships and entities into the Spring Boot API. The only thing we need to
be cognizant about is that each entity also has its own repository interface
and that they are correctly annotated with the JPA annotations.
