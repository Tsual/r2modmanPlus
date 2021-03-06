<template>
	<div>
		<hero title='Config editor' subtitle='Make changes to your configuration files' heroType='is-info'/>
		<div class='notification is-warning'>
			<p class='container'>Configuration files are generated after launching the game, with the mod installed, at
				least once.
			</p>
		</div>
		<div class='columns'>
			<div class='column is-full'>
				<article class='media'>
					<div class='media-content'>
						<div class='content'>
							<div class='container' v-if="editing === false">
								<div>
									<i class='fas fa-long-arrow-alt-left margin-right'/>
									<strong><a @click="backToManager()">Go back</a></strong>
									<br/><br/>
								</div>
                                <div class='card is-shadowless'>
                                    <div class='card-header-title'>

                                        <div class="input-group input-group--flex margin-right">
                                            <label for="local-search" class="non-selectable">Search</label>
                                            <input id="local-search" v-model='filterText' class="input margin-right" type="text" placeholder="Search for config files"/>
                                        </div>

                                        <div class="input-group margin-right">
                                            <label for="config-sort-order" class="non-selectable">Sort</label>
                                            <select id="config-sort-order" class="select select--content-spacing" v-model="sortOrder">
                                                <option v-for="(key, index) in getSortOrderOptions()" :key="`${index}-deprecated-position-option`">
                                                    {{key}}
                                                </option>
                                            </select>
                                            <span>&nbsp;</span>
                                            <select id="config-sort-direction" class="select select--content-spacing" v-model="sortDirection">
                                                <option v-for="(key, index) in getSortDirectionOptions()" :key="`${index}-deprecated-position-option`">
                                                    {{key}}
                                                </option>
                                            </select>
                                        </div>

                                    </div>
                                </div>
								<div v-for='(file, index) in sortedConfigFiles' :key="`config-file-${file.name}`">
									<expandable-card
											:id="index"
											:visible="false">
										<template v-slot:title>
											<span>{{file.name}}</span>
										</template>
										<a class='card-footer-item' @click="editFile(file)">Edit Config</a>
										<a class='card-footer-item' @click="deleteConfig(file.getPath())">Delete</a>
									</expandable-card>
								</div>
							</div>
							<div class='container' v-else-if="editing && loadedFile.getPath().toLowerCase().endsWith('.cfg')">
								<div>
									<div class='sticky-top sticky-top--buttons'>
										<button class='button is-info' @click="saveChanges()">Save changes</button>&nbsp;
										<button class='button is-danger' @click="editing = false; updateConfigList();">Cancel</button>
									</div>
									<br/>
									<h4 class='title is-4'>{{loadedFile.getName()}}</h4>
									<div>
										<h5 class='subtitle is-5'>Sections</h5>
										<ul>
											<li v-for='(_, key) in variables' :key="`section-link-${key}`">
												<a :href="`#section-${key}`" v-if="key.length > 0">{{key}}</a>
												<a :href="`#section-${key}`" v-else>[UNTITLED]</a>
											</li>
										</ul>
										<hr/>
										<div v-for='(vars, key) in variables' :key="`section-${key}`">
											<br/>
											<h5 :id="`section-${key}`" class='subtitle subtitle--is-background is-5 sticky-top'>[{{key}}]</h5>
											<div v-for='(varValue, varName) in vars' :key="`vars-${varName}`">
												<div class='field has-addons has-tooltip-top has-tooltip-multiline'
												     :data-tooltip="getCommentDisplay(varValue.comments).length > 0 ? getCommentDisplay(varValue.comments) : undefined">
													<div class='control is-expanded'>
														<input class='input' type='text' :value="varName" width="250"
														       disabled readonly/>
													</div>
													<div class='control is-expanded'>
														<input :id="`${key}-${varName}`" class='input' type='text'
														       :value="varValue.value"
														       @change="updateVariableText(key, varName, this)"/>
													</div>
													<hr class='hr'/>
												</div>
											</div>
										</div>
									</div>
								</div>
							</div>
                            <div class='container' v-else-if="editing">
                                <div class='sticky-top sticky-top--buttons'>
                                    <button class='button is-info' @click="saveFreeText">Save changes</button>&nbsp;
                                    <button class='button is-danger' @click="editing = false; updateConfigList();">Cancel</button>
                                </div>
                                <br/>
                                <h4 class='title is-4'>{{loadedFile.getName()}}</h4>
                                <QuillEditor v-model="fileText" @input="updateFreeText"/>
                            </div>
						</div>
					</div>
				</article>
			</div>
		</div>
	</div>
</template>

<script lang='ts'>
    import Vue from 'vue';
    import { Component, Watch } from 'vue-property-decorator';
    import { ExpandableCard, Hero } from '../components/all';

    import Profile from 'src/model/Profile';
    import ConfigFile from 'src/model/file/ConfigFile';
    import ConfigLine from 'src/model/file/ConfigLine';

    import { Logger, LogSeverity } from 'src/r2mm/logging/Logger';

    import * as path from 'path';
    import * as fs from 'fs-extra';
    import BepInExTree from '../model/file/BepInExTree';
    import R2Error from '../model/errors/R2Error';
    import { SortConfigFile } from '../model/real_enums/sort/SortConfigFile';
    import { SortDirection } from '../model/real_enums/sort/SortDirection';
    import ConfigSort from '../r2mm/configs/ConfigSort';

    import QuillEditor from '../components/QuillEditor.vue';

    @Component({
		components: {
			'hero': Hero,
			'expandable-card': ExpandableCard,
            QuillEditor
		}
	})
	export default class ConfigEditor extends Vue {

		private shownConfigFiles: ConfigFile[] = [];
		private configFiles: ConfigFile[] = [];
		private loadedFile: ConfigFile | null = null;
		private fileText: string = '';

		private editing: boolean = false;
		private variables: { [section: string]: { [variable: string]: ConfigLine } } = {};

		private filterText: string = '';
		private sortOrder: SortConfigFile = SortConfigFile.NAME;
		private sortDirection: SortDirection = SortDirection.STANDARD;

		@Watch('filterText')
		textChanged() {
			this.shownConfigFiles = this.configFiles.filter((conf: ConfigFile) => conf.getName().toLowerCase().match(this.filterText.toLowerCase()));
		}

		get sortedConfigFiles(): ConfigFile[] {
		    return ConfigSort.sort(this.shownConfigFiles, this.sortOrder, this.sortDirection);
        }

		editFile(file: ConfigFile) {
			const configLocation: string = path.join(Profile.getActiveProfile().getPathOfProfile(), 'BepInEx', 'config');
			this.loadedFile = file;
			this.fileText = fs.readFileSync(file.getPath()).toString();

			if (file.getPath().toLowerCase().endsWith(".cfg")) {
                // Find all variables offered within config script.
                this.variables = {};
                let section = 'root';
                let comments: string[] = [];
                this.fileText.split('\n').forEach((line: string) => {
                    if (line.trim().startsWith('[') && line.trim().endsWith(']')) {
                        section = line.trim().substring(1, line.trim().length - 1);
                        this.variables[section] = {};
                        comments = [];
                    } else if (!line.trim().startsWith('#') && line.search('=') > 0) {
                        const sides = line.split('=');
                        const rightSide = sides.splice(1).join("=");
                        this.variables[section][sides[0].trim()] = new ConfigLine(rightSide.trim(), comments);
                        comments = [];
                    } else if (line.trim().startsWith('#')) {
                        comments.push(line.trim());
                    }
                });
            }
			window.scrollTo(0, 0);
			this.editing = true;
		}

		saveChanges() {
			let builtString = '';
			let section = 'root';
			this.fileText.split('\n').forEach((line: string) => {
				if (line.trim().startsWith('[') && line.trim().endsWith(']')) {
					section = line.trim().substring(1, line.trim().length - 1);
					builtString += line + '\n';
				} else if (!line.trim().startsWith('#') && line.search('=') > 0) {
					const sides = line.split('=');
					builtString += `${sides[0].trim()} = ${this.variables[section][sides[0].trim()].getValue()}\n`;
				} else {
					builtString += line + '\n';
				}
			});
			fs.writeFileSync(this.loadedFile!.getPath(), builtString.trim());
			window.scrollTo(0, 0);
			this.editing = false;
            this.updateConfigList();
		}

		backToManager() {
			this.$router.push('/manager');
		}

		updateVariableText(section: string, variable: string) {
			const element: HTMLElement | null = document.getElementById(`${section}-${variable}`);
			if (element instanceof HTMLElement) {
				const inputField = element as HTMLInputElement;
				this.variables[section][variable].setValue(inputField.value);
			}
		}

		deleteConfig(fileName: string) {
			fs.removeSync(fileName);
            this.updateConfigList();
		}

        getCommentDisplay(comments: string[]): string {
            return comments.join("\n")
                .replace(new RegExp("#+", "g"), "")
                .trim();
        }

        getSortOrderOptions() {
		    return Object.values(SortConfigFile);
        }

        getSortDirectionOptions() {
		    return Object.values(SortDirection);
        }

		created() {
		    this.updateConfigList();
		}

		updateConfigList() {
		    this.configFiles = [];
            const configLocation: string = path.join(Profile.getActiveProfile().getPathOfProfile(), 'BepInEx', 'config');
            if (fs.pathExistsSync(configLocation)) {
                const tree: BepInExTree | R2Error = BepInExTree.buildFromLocation(configLocation);
                if (tree instanceof BepInExTree) {
                    tree.getRecursiveFiles().forEach(file => {
                        if (path.extname(file).toLowerCase() === '.cfg' || path.extname(file).toLowerCase() === '.txt' || path.extname(file).toLowerCase() === '.xml') {
                            const fileStat = fs.lstatSync(file);
                            this.configFiles.push(new ConfigFile(file.substring(configLocation.length + 1, file.length - 4), file, fileStat.mtime));
                        } else if (path.extname(file).toLowerCase() === '.json') {
                            const fileStat = fs.lstatSync(file);
                            this.configFiles.push(new ConfigFile(file.substring(configLocation.length + 1, file.length - 5), file, fileStat.mtime));
                        }
                    });
                } else {
                    Logger.Log(LogSeverity.ACTION_STOPPED, `${tree.name}\n-> ${tree.message}`);
                }
            }
            this.textChanged();
        }

        updateFreeText(text: string) {
		    this.fileText = text;
        }

        saveFreeText() {
            fs.writeFileSync(this.loadedFile!.getPath(), this.fileText);
            window.scrollTo(0, 0);
            this.editing = false;
            this.updateConfigList();
        }

	}
</script>
